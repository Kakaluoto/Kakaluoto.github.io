---
title: 【Linux】基于ptrace系统调用实现一个debugger
date: 2022-2-22
tags: [Linux,C++]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212172023986.webp
mathjax: true
---
# 基于ptrace的debugger设计
## 1. 程序的设计思路
### 1.1 设计思路

本次设计实现的debugger针对被调试进程主要实现了6项功能:
+ 可以读取被调试进程CPU所有寄存器的值
+ 可以对被调试进程进行单步调试
+ 可以恢复被调试进程运行
+ 可以查看被调试进程任意内存空间
+ 可以计算被调试进程执行完需要多少条指令
+ 可以在指定地址插入断点

为了在不同的功能之间进行切换，使用循环轮询手动输入参数的方式来决定使用哪一项功能。

```cpp
Type "exit" to exit debugger.
Type "reg" or "r" to show registers.
Type "step" or "s" to single step.
Type "continue" or "c" to continue until tracee stop.
Type "memory" or "m" to show memory content.
	You can use "-addr" or "-off" or "-nb" as argument.
	use "-addr" to specify hexadecimal start address of the memory
		for example: Type "m -addr ff" to specify the start address 0xff
		(default start address is RIP)
	use "-off" to specify the decimal offset from the start address
		(default offset is 0)
	use "-nb" to specify the decimal number of bytes to be displayed
		(default number is 40)
Type "ic" to count total instructions.
Type "break" or "b" to insert breakpoint.
	for example: Type "b 555555555131" to specify the breakpoint address 0x555555555131
```

系统调用Ptrace的定义：

```c++
long ptrace(enum __ptrace_request request, pid_t pid,void *addr,void *data);
```

ptrace的第一个参数可以通过指定request请求来实现不同的功能。使用PTRACE_GETREGS参数来一次性获取所有寄存器的值，使用PTRACE_SINGLESTEP来进行单步调试，PTRACE_CONT来让被暂停的进程恢复运行。

为了读取任意内存空间，需要知道内存空间的起始地址，一次性读取多少个字节，因此默认采用rip寄存器存放的指针作为默认的起始地址，也就是默认从下一条指令的地址开始读，可以指定一次性读多少个字节，这里我默认一次性读取40个字节，为了既能够读到rip指针之后的数据也能读到rip指针之前的数据，引入偏移量offset，这样可以在指定了起始地址的基础上加上偏移量，从而理论上能够读取任意内存区域。当然，如果明确知道要读的内存起始地址，也可以忽略rip指针直接指定起始地址。

计算进程执行完需要多少条指令比较简单，只需要不停单步执行直到退出，每执行一步就计数即可。

给进程打断点的实现最为困难，本次设计仅针对进程特定地址进行插入断点。可以使用Ptrace的PTRACE_PEEKDATA，PTRACE_POKEDATA两个请求，来在进程指定的地址读出指令和注入新的指令。因此可以在指定的地址插入int3(0xcc)中断指令实现断点，为了让插入断点的进程依然能够恢复运行，在插入断点之前对该地址原有指令进行备份，遇到断点之后再将备份的指令还原，并且恢复命中断点时的寄存器值，尤其是rip指针需要减1，回退一个地址。

![](https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171713946.png)

过程如上图所示，第一步rip先指向byte2对应地址处，利用PTRACE_PEEKDATA将byte2,byte3取出备份，同时保存当前寄存器值，为恢复做备份。第二步插入0xcc,0x00指令，即int3中断指令，执行一步来到第三步rip指向0x00，触发中断，子进程暂停。第四步，为了让子进程继续运行，将备份的原始指令写入rip-1处，并且利用PTRACE_SETREGS将寄存器值恢复成原来的值，此时rip跟着上移。这样子进程可以继续正常运行不会core dump。以上四步构成了在byte2对应地址处打上断点的操作。

要完成插入断点并且运行到断点停止，并且能恢复原有指令继续正常运行的非常关键的一点就是需要知道子进程是否命中断点。因为子进程完全有可能因为接收到其他信号而暂停，同时产生SIGTRAP信号发送给父进程，并不一定就是因为断点而暂停并发送SIGTRAP信号。因此在等待被调试进程的时候，当截获SIGTRAP信号需要取出rip指针，此时如果是断点触发的暂停信号，rip肯定指向0xcc指令的下一条指令，故而只需要判断当初我们输入的打断点的地址addr是否等于rip-1。如果相等那么断点命中，命中之后就可以将原有指令恢复，把寄存器值恢复。

## 2. 程序的模块划分

主要函数

```Cpp
void getdata(pid_t child, long addr, char* str, int len);
/* *
 * 从子进程指定地址插入数据
 * child: 子进程pid号
 * addr: 地址
 * str: 用来插入的字节
 * len: 插入字节数
 * */

void putdata(pid_t child, long addr, char* str, int len);
/* *
 * 按字节打印数据
 * tip: 可以附带 字符串输出
 * codes: 需要打印的字节
 * len: 需要打印的字节数
 * */

void showMemory(pid_t pid, 
                unsigned long long addr, long offset = 0, int nbytes = 40);
/* *
 * 显示任意内存内容
 * pid: 子进程pid
 * addr: 指定内存基地址
 * offset: 指定相对于基地址的偏移地址
 * nbytes: 需要显示的字节数
 * */

int wait_breakpoint(pid_t pid, int status, Breakpoint& bp);
/* *
 * 注入断点
 * pid: 子进程pid
 * bp: 断点结构体
 struct Breakpoint {
    unsigned long long addr;
    char backup[CODE_SIZE];
    bool breakpoint_mode;
};
//断点结构体，需要插入断点的地址addr
//断点地址处的指令的备份backup
//用来标记是否有断点存在的变量breakpoint_mode
 * */

void breakpoint_inject(pid_t pid, Breakpoint& bp);
/* *
 * 等待断点，判断是否命中
 * pid: 子进程pid
 * status: 由外部传入，获取当前tracee停止的状态码
 * bp: 断点结构体
 * */

void get_base_address(pid_t pid, unsigned long long& base_addr);
/* *
 * 获取子进程再虚拟地址空间的起始地址
 * pid: 子进程pid
 * base_addr: 用来存储起始地址
 * */

void show_help();
//显示帮助信息
```



## 3. 遇到的问题及解决方法
### 3.1 Linux地址空间随机化产生的问题

运行代码fork子进程之后，循环单步执行，每执行一步输出一次rip指针，共万步有余。每次执行代码输出的rip都各不相同，从第一次输出的rip到最后一次输出的rip指针并不是固定的几个值，而是每次执行输出的都是一批不同的rip序列。这给后期断点功能的实现造成了很大的麻烦。比如我使用GDB给被调试进程main函数第一行打断点，执行到断点处执行`i r rip`命令观察此时rip值，假设此时rip的值为aaa。我以为获得了子进程源代码main函数第一行的地址即aaa，于是将其设为断点地址却发现断点命中不了。

为了确认我自己代码fork出的子进程所有指令的地址里有aaa，我单步执行，每次单步就取一次rip指针的值，与aaa进行比对，发现没有任何地址与aaa相等，这与gdb给出的结果不符。且每次运行，rip输出的序列内容都和上一次运行输出的rip序列不同。经过查找资料确定是Linux地址空间随机化的缘故。ASLR 技术将进程的某些内存空间地址进行随机化来增大入侵者预测目的地址的难度，从而降低进程被成功入侵的风险。

使用命令`sudo bash -c "echo 0 > /proc/sys/kernel/randomize_va_space"`关闭了ASLR,之后rip输出的序列不再随机变化，而是固定的序列。由此GDB获取到的rip地址和我自己获取的rip开始保持一致。

### 3.2 无法确定应该注入断点的地址

解决了被调试进程虚拟地址总是变化的问题之后，就可以指定断点的地址了。使用反汇编命令`objdump -d test`获得被调试子进程的汇编代码以及每条汇编代码的偏移地址。我发现gdb断点到相应的行给出来的地址和反汇编的地址不一样，这样如果想要通过反汇编找到main函数入口地址，根据这个入口地址设置断点是无法成功的，通过观察我发现反汇编出来的地址是偏移地址，这个偏移地址总是与被调试进程对应指令的实际虚拟地址相差一个常数，我这里是0x5555_5555_4000。比如反汇编被调试子进程main函数入口地址是0x1129，直接将0x1129作为断点地址会报错，如果将0x5555_5555_4000 + 0x1129作为main函数的虚拟地址就可以断点注入成功，而且这个相加的和与gdb获得的地址一致。

本来直接将这个神秘常数拿来相加就可以利用反汇编得到的地址打断点了，但是我不能保证所有的被调试进程都是相差这个常数，经过查阅资料我知道0x5555_5555_4000是子进程的虚拟地址空间的首地址，通过`pmap -x pid`命令可以获取任意进程的内存分布范围。查阅资料得知，Linux将进程的内存分布信息缓存在/proc/进程pid/maps文件中，pmap的原理也是解析这个文件，于是我通过解析这个文件便成功获取到了子进程的虚拟内存起始地址。

如此就可以很方便地通过反汇编`objdump -d`指令获取汇编的偏移地址，作为断点地址的参数进行断点注入了，而无需关心子进程的虚拟内存其实地址是多少，因为反汇编得出来的汇编指令的地址是不变的。

### 3.3 断点命中成功，恢复源代码失败

恢复寄存器的时候忘记调整rip指针了，应该将rip指针减一，回退到断点的地址处。

## 4. 程序使用说明及运行结果

当前目录下含有5个文件

```shell
/ptrace_debugger$ tree
.
├── ASLR.sh
├── main.cpp
├── ptrace_debugger
├── test
└── test.cpp

0 directories, 5 files

```

Linux 平台上 ASLR 分为 0，1，2 三级，用户可以通过一个内核参数 randomize_va_space 进行等级控制。它们对应的效果如下：

0：没有随机化。即关闭 ASLR。
1：保留的随机化。共享库、栈、mmap() 以及 VDSO 将被随机化。
2：完全的随机化。在 1 的基础上，通过 brk() 分配的内存空间也将被随机化。

ASLR.sh脚本用来设置随机化等级：

ptrace_debugger是main.cpp编译的可执行文件

test是被调试进程test.cpp编译的可执行文件

执行如下命令关闭随机化：

```shell
/ptrace_debugger$ ./ASLR.sh 0
change ASLR level to:
0
```

运行ptrace_debugger：

```shell
/ptrace_debugger$ ./ptrace_debugger 
This is a debugger based on ptrace.
For help type "help" or "h"
Please input the name of program to be traced:
test
(PDebugger) >
```

查看寄存器：

```shell
(PDebugger) >r 
rax	0
rbx	0
rcx	0
rdx	0
rsi	0
rdi	0
rbp	0
rsp	7fffffffdf50
rip	7ffff7fd0100
eflags	200
cs	33
ss	2b
ds	0
es	0
(PDebugger) >
```

单步调试：

```shell
(PDebugger) >r
rax	0
rbx	0
rcx	0
rdx	0
rsi	0
rdi	0
rbp	0
rsp	7fffffffdf50
rip	7ffff7fd0100
eflags	200
cs	33
ss	2b
ds	0
es	0
(PDebugger) >s
(PDebugger) >r
rax	0
rbx	0
rcx	0
rdx	0
rsi	0
rdi	7fffffffdf50
rbp	0
rsp	7fffffffdf50
rip	7ffff7fd0103
eflags	202
cs	33
ss	2b
ds	0
es	0
(PDebugger) >
```

恢复运行：

```shell
(PDebugger) >s
(PDebugger) >r
rax	0
rbx	0
rcx	0
rdx	0
rsi	0
rdi	7fffffffdf50
rbp	0
rsp	7fffffffdf48
rip	7ffff7fd0df0
eflags	202
cs	33
ss	2b
ds	0
es	0
(PDebugger) >c
Process finished.
```

查看任意内存空间：

```shell
(PDebugger) >m -off -20 -nb 40
current base address is : 0x7ffff7fd0df0
offset is : -20
The 40 bytes after start address: 0x7ffff7fd0ddc :
00 00 00 00 bf 01 00 00 
00 5b e9 95 d4 01 00 0f 
1f 44 00 00 f3 0f 1e fa 
55 48 89 e5 41 57 49 89 
ff 41 56 41 55 41 54 53 

(PDebugger) >
```

计算指令数：

```shell
(PDebugger) >ic

total instruction count is 117802

```

断点调试：

先进行反汇编

```shell
hy@ubuntu:~/下载/ptrace_debugger$ ls
ASLR.sh  main.cpp  ptrace_debugger  test  test.cpp
hy@ubuntu:~/下载/ptrace_debugger$ objdump -d test

test：     文件格式 elf64-x86-64

   
......省略......


0000000000001129 <main>:
    1129:	f3 0f 1e fa          	endbr64 
    112d:	55                   	push   %rbp
    112e:	48 89 e5             	mov    %rsp,%rbp
    1131:	c7 45 f4 04 00 00 00 	movl   $0x4,-0xc(%rbp)
    1138:	c7 45 f8 08 00 00 00 	movl   $0x8,-0x8(%rbp)
    113f:	8b 55 f4             	mov    -0xc(%rbp),%edx
    1142:	8b 45 f8             	mov    -0x8(%rbp),%eax
    1145:	01 d0                	add    %edx,%eax
    1147:	89 45 fc             	mov    %eax,-0x4(%rbp)
    114a:	b8 00 00 00 00       	mov    $0x0,%eax
    114f:	5d                   	pop    %rbp
    1150:	c3                   	retq   
    1151:	66 2e 0f 1f 84 00 00 	nopw   %cs:0x0(%rax,%rax,1)
    1158:	00 00 00 
    115b:	0f 1f 44 00 00       	nopl   0x0(%rax,%rax,1)

......省略......
 

Disassembly of section .fini:

00000000000011d8 <_fini>:
    11d8:	f3 0f 1e fa          	endbr64 
    11dc:	48 83 ec 08          	sub    $0x8,%rsp
    11e0:	48 83 c4 08          	add    $0x8,%rsp
    11e4:	c3                   	retq  
```

可以看到main函数入口地址是0x1129

打断点：

```shell
Please input the name of program to be traced:
test
(PDebugger) >b 1129
get base_addr:0x555555554000
get tracee instruction: f3 0f 1e fa 55 48 89 e5 

try to set breakpoint
set breakpoint instruction: cc 00 00 00 00 00 00 00 

(PDebugger) >c
Hit Breakpoint at: 0x555555555129
(PDebugger) >r
rax	555555555129
rbx	555555555160
rcx	555555555160
rdx	7fffffffdf68
rsi	7fffffffdf58
rdi	1
rbp	0
rsp	7fffffffde68
rip	555555555129
eflags	246
cs	33
ss	2b
ds	0
es	0
(PDebugger) >s
(PDebugger) >c
Process finished.
```

## 5. 代码

完整项目地址[GitHub - Kakaluoto/ptraceDebugger: 利用ptrace系统调用实现的debugger](https://github.com/Kakaluoto/ptraceDebugger)

### 5.1 被调试子进程tracee

test.cpp:

```cpp
int main() {
    int i = 4;
    int j = 8;
    int k = i + j;
    return 0;
}
```

### 5.2 关闭ASLR脚本

```shell
#!/bin/bash

if [ $# == 0 ]		# $# means the number of parameters
then
    echo 'current ASLR level:'
    cat /proc/sys/kernel/randomize_va_space
    echo 'use option "-h" for help.'
elif [ $# == 1 ]
then
    if [ $1 == 0 ]
    then 
        sudo bash -c "echo 0 > /proc/sys/kernel/randomize_va_space"
        echo "change ASLR level to:"
        cat /proc/sys/kernel/randomize_va_space
    elif [ $1 == 1 ]
    then
        sudo bash -c "echo 1 > /proc/sys/kernel/randomize_va_space"
        echo "change ASLR level to:"
        cat /proc/sys/kernel/randomize_va_space
    elif [ $1 == 2 ]
    then
        sudo bash -c "echo 2 > /proc/sys/kernel/randomize_va_space"
        echo "change ASLR level to:"
        cat /proc/sys/kernel/randomize_va_space
    elif [ $1 == "-h" ]
    then
        echo ""
        echo "### bash ./ASLR"
        echo "-->   show current ASLR level."
        echo ""
        echo "### bash ./ASLR -h"
        echo "-->   show help info."
        echo ""
        echo "### bash ./ASLR 0"
        echo "-->   change ASLR level to 0."
        echo ""
        echo "### bash ./ASLR 1"
        echo "-->   change ASLR level to 1."
        echo ""
        echo "### bash ./ASLR 2"
        echo "-->   change ASLR level to 2."
        echo ""
    else
        echo "syntax error!"
        echo 'use option "-h" for help.'
    fi
else
    echo "syntax error!"
    echo 'use option "-h" for help.'
fi
```

### 5.3 Debugger代码

```cpp
#include <iostream>
#include <vector>
#include <cstdio>
#include <cstdlib>
#include <cstring>
#include <unistd.h>
#include <sys/wait.h>
#include <sys/ptrace.h>
#include <sys/reg.h>
#include <sys/user.h>
#include <fstream>

#define LONG_SIZE 8 //LONG型数据的长度8个字节
#define CODE_SIZE 8//注入断点中断指令的长度，也是8个字节
using namespace std;
vector<string> argv;//存储当前命令所有参数
string cmd;//当前命令字符串
struct Breakpoint {
    unsigned long long addr;
    char backup[CODE_SIZE];
    bool breakpoint_mode;
};

//断点结构体，包含有需要插入断点的地址，对断点地址处的指令进行备份，以及用来标记是否有断点存在的变量
void argparse(); //解析参数

void getdata(pid_t child, long addr, char* str, int len);//从子进程指定地址获取指定长度的数据，长度单位为字节

void putdata(pid_t child, long addr, char* str, int len);//将数据插入子进程指定地址处

void printBytes(const char* tip, char* codes, int len);//打印字节

void showMemory(pid_t pid, unsigned long long addr, long offset = 0, int nbytes = 40);//显示指定地址处指定长度的内存内容

int wait_breakpoint(pid_t pid, int status, Breakpoint& bp);//判断断点是否命中

void breakpoint_inject(pid_t pid, Breakpoint& bp);//给子进程注入断点

void get_base_address(pid_t pid, unsigned long long& base_addr);//从当前子进程的虚拟地址范围获取子进程的起始地址

void show_help();//显示帮助信息

int main() {
    pid_t pid;
    string tracee_name;
    unsigned long long base_addr;
    printf("This is a debugger based on ptrace.\n"
           "For help type \"help\" or \"h\"\n");
    printf("Please input the name of program to be traced:\n");
    getline(cin, tracee_name);//获取本目录下被trace的进程
    tracee_name = "./" + tracee_name;//转换成路径
    int status;
    Breakpoint breakpoint = {.breakpoint_mode=false};//默认不进入断点模式
    switch (pid = fork()) {//fork子进程
        //fork子进程失败
        case -1:
            cout << "Failed to create subprocess!\n";
            return 0;
            //处理子进程
        case 0:
            if (ptrace(PTRACE_TRACEME, 0, nullptr, nullptr) < 0) {
                cout << "ptrace error in subprocess!\n";
                exit(1);
            }
            if (execl(tracee_name.data(), tracee_name.data())) {
                cout << "execvp error in subprocess!\n";
                exit(2);
            }
            //子进程，没有成功执行
            cout << "invalid input command : \"" << tracee_name << "\"" << endl;
            exit(3);
        default: {
            while (true) {//开始轮询输入的命令
                printf("(PDebugger) >");
                getline(cin, cmd);
                // 如果输入为exit 则结束当前进程
                if (strcmp(cmd.data(), "exit") == 0) {
                    break;
                }
                argparse();//输入参数解析
                //execute_cmd(pid);
                struct user_regs_struct regs{};//存储子进程当前寄存器的值
                int argc = argv.size();
                char** arguments = new char* [argc];//转换参数类型，以便能够喂到exec函数
                for (int i = 0; i < argc; i++) {
                    arguments[i] = (char*) argv[i].data();
                }
                if (strcmp(arguments[0], "exit") == 0) {//退出操作
                    ptrace(PTRACE_KILL, pid, nullptr, nullptr);//杀死子进程，避免出现僵尸进程
                    break;
                } else if (strcmp(arguments[0], "reg") == 0 || strcmp(arguments[0], "r") == 0) {//获取寄存器内容
                    ptrace(PTRACE_GETREGS, pid, nullptr, &regs);
                    printf("rax\t%llx\nrbx\t%llx\nrcx\t%llx\nrdx\t%llx\nrsi\t%llx\nrdi\t%llx\nrbp\t%llx\n"
                           "rsp\t%llx\nrip\t%llx\neflags\t%llx\ncs\t%llx\nss\t%llx\nds\t%llx\nes\t%llx\n",
                           regs.rax, regs.rbx, regs.rcx, regs.rdx, regs.rsi, regs.rdi, regs.rbp,
                           regs.rsp, regs.rip, regs.eflags, regs.cs, regs.ss, regs.ds, regs.es);
                } else if (strcmp(arguments[0], "step") == 0 || strcmp(arguments[0], "s") == 0) {//单步调试
                    ptrace(PTRACE_SINGLESTEP, pid, nullptr, nullptr);//发送single step给子进程
                    wait(&status);//等待子进程收到sigtrap信号
                    if (WIFEXITED(status)) {//执行到最后一条指令退出循环，同时父进程也会结束
                        printf("Process finished.\n");
                        break;
                    }
                } else if (strcmp(arguments[0], "continue") == 0 || strcmp(arguments[0], "c") == 0) {//继续执行
                    ptrace(PTRACE_CONT, pid, nullptr, nullptr);//继续执行，一直到子进程发出发出暂停信号
                    wait(&status);//等待子进程停止，并获取子进程状态值
                    if (!breakpoint.breakpoint_mode) {//没有断点，一直执行到子进程结束
                        if (WIFEXITED(status)) {
                            printf("Process finished.\n");
                            exit(0);
                        }
                    } else {//断点模式被激活，breakpoint_mode字段被置为true
                        wait_breakpoint(pid, status, breakpoint);//等待并判断断点是否被命中
                    }
                } else if (strcmp(arguments[0], "memory") == 0 || strcmp(arguments[0], "m") == 0) {//获取子进程制定区域的内存内容
                    ptrace(PTRACE_GETREGS, pid, nullptr, &regs);
                    struct Params {//默认地址采用rip指针的内容，偏移默认为0，默认读取40个字节
                        unsigned long long addr;
                        long offset;
                        int nbytes;
                    } params = {regs.rip, 0, 40};
                    if (argc == 1) {
                        showMemory(pid, regs.rip);//显示内存内容
                    } else {
                        for (int i = 1; i < argc; i++) {//检查是否有额外参数指定
                            if (strcmp(arguments[i], "-addr") == 0) {//指定内存的起始地址
                                params.addr = strtol(arguments[++i], nullptr, 16);
                                continue;//当前参数指定功能，下一个参数指定具体的值，两项获取之后直接跳一步检查别的参数
                            }
                            if (strcmp(arguments[i], "-off") == 0) {
                                params.offset = strtol(arguments[++i], nullptr, 10);
                                continue;
                            }
                            if (strcmp(arguments[i], "-nb") == 0) {
                                params.nbytes = strtol(arguments[++i], nullptr, 10);
                                continue;
                            }
                        }
                        showMemory(pid, params.addr, params.offset, params.nbytes);
                    }
                } else if (strcmp(arguments[0], "ic") == 0) {//计算执行完毕所需指令数
                    long count = 0;
//                    struct user_regs_struct temp_regs{};//存储子进程当前寄存器的值
                    while (true) {
                        wait(&status);//当前子进程还是暂停状态，父进程被阻塞
                        if (WIFEXITED(status)) {
                            printf("\ntotal instruction count is %ld\n", count);
                            exit(0);//指令执行完子进程也结束运行了，父进程退出
                        }
                        ptrace(PTRACE_SINGLESTEP, pid, nullptr, nullptr);//单步执行下一条指令
//                        ptrace(PTRACE_GETREGS, pid, nullptr, &temp_regs);
//                        printf("RIP:%llx\t", temp_regs.rip);
                        count++;
                    }
                } else if (strcmp(arguments[0], "break") == 0 || strcmp(arguments[0], "b") == 0) {
                    if (argc == 2) {//打断点
                        get_base_address(pid, base_addr);//获取子进程的起始虚拟地址
                        //输入的地址实际上是利用objdump反汇编得到的偏移地址，相加得到在虚拟内存中的实际地址
                        breakpoint.addr = strtol(arguments[1], nullptr, 16) + base_addr;
                        breakpoint_inject(pid, breakpoint);//注入断点
                    } else {
                        printf("Please input the address of breakpoint!\n");
                    }
                } else if (strcmp(arguments[0], "help") == 0 || strcmp(arguments[0], "h") == 0) {
                    show_help();//显示帮助信息
                } else {
                    cout << "Invalid Argument!\n";
                }
                argv.clear();//下一轮参数输入之前需要把当前存储的命令清除
            }
            wait(&status);//等待子进程结束之后父进程再退出
        }
    }
}

void argparse() {//解析输入参数
    string param;
    for (char i:cmd + " ") {//因为要用到空格进行分割，为了防止最后一个参数分割不到加一个空格
        if (i != ' ') {
            param += i;
        } else {
            argv.push_back(param);
            param = "";
            continue;
        }
    }
}

/* *
 * 从子进程指定地址读取数据
 * child: 子进程pid号
 * addr: 地址
 * str: 用来存储读取的字节
 * len: 读取字节长度
 * */
void getdata(pid_t child, unsigned long long addr, char* str, int len) {
    char* laddr = str;
    int i = 0, j = len / LONG_SIZE;//计算一共需要读取多少个字
    union u {
        long val;
        char chars[LONG_SIZE];
    } word{};
    while (i < j) {//每次读取1个字，8个字节，每次地址加8(LONG_SIZE)
        word.val = ptrace(PTRACE_PEEKDATA, child, addr + i * LONG_SIZE, nullptr);
        if (word.val == -1)
            perror("trace error");
        memcpy(laddr, word.chars, LONG_SIZE);//将这8个字节拷贝进数组
        ++i;
        laddr += LONG_SIZE;
    }
    j = len % LONG_SIZE;//不足一个字的虚读一个字
    if (j != 0) {
        word.val = ptrace(PTRACE_PEEKDATA, child, addr + i * LONG_SIZE, nullptr);
        if (word.val == -1)
            perror("trace error");
    }
    str[len] = '\0';
}

/* *
 * 从子进程指定地址插入数据
 * child: 子进程pid号
 * addr: 地址
 * str: 用来插入的字节
 * len: 插入字节数
 * */
void putdata(pid_t child, unsigned long long addr, char* str, int len) {
    char* laddr = str;//与getdata类似
    int i = 0, j = len / LONG_SIZE;
    union u {
        long val;
        char chars[LONG_SIZE];
    } word{};
    while (i < j) {
        memcpy(word.chars, laddr, LONG_SIZE);
        if (ptrace(PTRACE_POKEDATA, child, addr + i * LONG_SIZE, word.val) == -1)
            perror("trace error");
        ++i;
        laddr += LONG_SIZE;
    }
    j = len % LONG_SIZE;
    if (j != 0) {
        word.val = 0;
        memcpy(word.chars, laddr, j);
        if (ptrace(PTRACE_POKEDATA, child, addr + i * LONG_SIZE, word.val) == -1)
            perror("trace error");
    }
}

/* *
 * 按字节打印数据
 * tip: 可以附带 字符串输出
 * codes: 需要打印的字节
 * len: 需要打印的字节数
 * */
void printBytes(const char* tip, char* codes, int len) {
    int i;
    printf("%s", tip);
    for (i = 0; i < len; ++i) {
        printf("%02x ", (unsigned char) codes[i]);
        if ((i + 1) % 8 == 0)
            printf("\n");
    }
    puts("");
}

/* *
 * 显示任意内存内容
 * pid: 子进程pid
 * addr: 指定内存基地址
 * offset: 指定相对于基地址的偏移地址
 * nbytes: 需要显示的字节数
 * */
void showMemory(pid_t pid, unsigned long long addr, long offset, int nbytes) {
    printf("current base address is : 0x%llx\n"//显示任意内存内容
           "offset is : %ld\n", addr, offset);
    auto* memory_content = new char[nbytes];
    getdata(pid, addr + offset, memory_content, nbytes);//从指定的地址按照指定的偏移量读取指定的字节数
    printf("The %d bytes after start address: 0x%llx :\n", nbytes, addr + offset);
    printBytes("", memory_content, nbytes);
}

/* *
 * 注入断点
 * pid: 子进程pid
 * bp: 断点结构体
 * */
void breakpoint_inject(pid_t pid, Breakpoint& bp) {
    char code[LONG_SIZE] = {static_cast<char>(0xcc)};//int3中断指令
    //copy instructions into backup variable
    getdata(pid, bp.addr, bp.backup, CODE_SIZE);//先把需要打断点的地址上指令取出备份
    printBytes("get tracee instruction: ", bp.backup, LONG_SIZE);
    puts("try to set breakpoint");
    printBytes("set breakpoint instruction: ", code, LONG_SIZE);
    putdata(pid, bp.addr, code, CODE_SIZE);//将中断指令int3注入
    bp.breakpoint_mode = true;//将断点模式标识变量置为true
}

/* *
 * 等待断点，判断是否命中
 * pid: 子进程pid
 * status: 由外部传入，获取当前tracee停止的状态码
 * bp: 断点结构体
 * */
int wait_breakpoint(pid_t pid, int status, Breakpoint& bp) {
    struct user_regs_struct regs{};
    /* 捕获信号之后判断信号类型	*/
    if (WIFEXITED(status)) {
        /* 如果是EXit信号 */
        printf("\nsubprocess EXITED!\n");
        exit(0);
    }
    if (WIFSTOPPED(status)) {
        /* 如果是STOP信号 */
        if (WSTOPSIG(status) == SIGTRAP) {                //如果是触发了SIGTRAP,说明碰到了断点
            ptrace(PTRACE_GETREGS, pid, 0, &regs);    //读取此时用户态寄存器的值，准备为回退做准备
            /* 将此时的指针与我的addr做对比，如果满足关系，说明断点命中 */
            if (bp.addr != (regs.rip - 1)) {
                /*未命中*/
                printf("Miss, fail to hit, rip:0x%llx\n", regs.rip);
                return -1;
            } else {
                /*如果命中*/
                printf("Hit Breakpoint at: 0x%llx\n", bp.addr);
                /*把INT 3 patch 回本来正常的指令*/
                putdata(pid, bp.addr, bp.backup, CODE_SIZE);
                ptrace(PTRACE_SETREGS, pid, nullptr, &regs);
                /*执行流回退，重新执行正确的指令*/
                regs.rip = bp.addr;//addr与rip不相等，恢复时以addr为准
                ptrace(PTRACE_SETREGS, pid, 0, &regs);
                bp.breakpoint_mode = false;//命中断点之后取消断点状态
                return 1;
            }
        }
    }
    return 0;
}

/* *
 * 获取子进程再虚拟地址空间的起始地址
 * pid: 子进程pid
 * base_addr: 用来存储起始地址
 * */
void get_base_address(pid_t pid, unsigned long long& base_addr) {
    /* *
     * Linux将每一个进程的内存分布暴露出来，以供读取
     * 每个进程的内存分布文件放在/proc/进程pid/maps文件夹里
     * 通过获取pid来读取对应的maps文件
     * */
    string memory_path = "/proc/" + to_string(pid) + "/maps";
    ifstream inf(memory_path.data());//建立输入流
    if (!inf) {
        cerr << "read failed!\n";
        return;
    }
    string line;
    getline(inf, line);//读第一行，根据文件的特点，起始地址之后是"-"字符
    base_addr = strtol(line.data(), nullptr, 16);//默认读到"-"字符为止，16进制
    cout << "get base_addr:0x" << hex << base_addr << endl;
}

void show_help() {
    printf("Type \"exit\" to exit debugger.\n");
    printf("Type \"reg\" or \"r\" to show registers.\n");
    printf("Type \"step\" or \"s\" to single step.\n");
    printf("Type \"continue\" or \"c\" to continue until tracee stop.\n");
    printf("Type \"memory\" or \"m\" to show memory content.\n"
           "\tYou can use \"-addr\" or \"-off\" or \"-nb\" as argument.\n"
           "\tuse \"-addr\" to specify hexadecimal start address of the memory\n"
           "\t\tfor example: Type \"m -addr ff\" to specify the start address 0xff\n"
           "\t\t(default start address is RIP)\n"
           "\tuse \"-off\" to specify the decimal offset from the start address\n"
           "\t\t(default offset is 0)\n"
           "\tuse \"-nb\" to specify the decimal number of bytes to be displayed\n"
           "\t\t(default number is 40)\n");
    printf("Type \"ic\" to count total instructions.\n");
    printf("Type \"break\" or \"b\" to insert breakpoint.\n"
           "\tfor example: Type \"b 555555555131\" to specify the breakpoint address 0x555555555131\n");
}
```



## 参考文章                                          

[linux工具pmap原理？ - 知乎](https://www.zhihu.com/question/60558543)
[linux - 在 Linux 中如何确定 PIE 可执行文件的文本部分的地址？ - IT工具网](https://www.coder.work/article/6731993)
[Linux虚拟地址空间布局以及进程栈和线程栈总结 - Xzzzh - 博客园](https://www.cnblogs.com/xzzzh/p/6596982.html)
[Linux虚拟地址空间布局 - clover_toeic - 博客园](https://www.cnblogs.com/clover-toeic/p/3754433.html)
[针对 Linux 环境下 gdb 动态调试获取的局部变量地址与直接运行程序时不一致问题的解决方案 - yhjoker - 博客园](https://www.cnblogs.com/yhjoker/p/9161716.html)
[Writing a Linux Debugger Part 3: Registers and memory](https://blog.tartanllama.xyz/writing-a-linux-debugger-registers/)
[浅析Linux 64位系统虚拟地址和物理地址的映射及验证方法 - Binfun - 博客园](https://www.cnblogs.com/binfun/p/14175659.html)
[Linux：断点原理与实现 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1626930)
[一口气看完45个寄存器，CPU核心技术大揭秘 - 知乎](https://zhuanlan.zhihu.com/p/272135463)
[断点原理与实现 - 知乎](https://zhuanlan.zhihu.com/p/110793460)
[一窥GDB原理](https://mp.weixin.qq.com/s/teERWh9IRMuO6tieOZNNng)
[Linux沙箱入门——ptrace从0到1 - 安全客，安全资讯平台](https://www.anquanke.com/post/id/231078#h3-3)
[Linux内核学习笔记（4）-- wait、waitpid、wait3 和 wait4 - tongye - 博客园](https://www.cnblogs.com/tongye/p/9558320.html)
[GDB原理之ptrace实现原理 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1742878)
[用图文带你彻底弄懂GDB调试原理 - 云+社区 - 腾讯云](https://cloud.tencent.com/developer/article/1823078?from=article.detail.1823077)
[Linux Hook 笔记 - evilpan](https://evilpan.com/2016/02/22/linux-hook-ptrace/)
[[原创]一窥GDB原理-Pwn-看雪论坛-安全社区|安全招聘|bbs.pediy.com](https://bbs.pediy.com/thread-265599.htm)
[Linux沙箱入门——ptrace从0到1 - 安全客，安全资讯平台](https://www.anquanke.com/post/id/231078)
[Linux信号列表及其详解 - zy010101 - 博客园](https://www.cnblogs.com/zy666/p/10504272.html)
[Linux 信号（signal） - 简书](https://www.jianshu.com/p/f445bfeea40a)
[PTRACE - Linux手册页-之路教程](https://www.onitroad.com/jc/linux/man-pages/linux/man2/ptrace.2.html)
[Linux的中断和系统调用 & esp、eip等寄存器 - blcblc - 博客园](https://www.cnblogs.com/charlesblc/p/6434321.html)
[Linux ptrace 简介](https://gohalo.me/post/linux-ptrace-api-introduce.html)
[Linux Hook 笔记 - 有价值炮灰 - 博客园](https://www.cnblogs.com/pannengzhi/p/5203467.html)
[linux ptrace II - mmmmar - 博客园](https://www.cnblogs.com/mmmmar/p/6048711.html)
[Ptrace--Linux中一种代码注入技术的应用 | 力托斯特的博客](https://0litost0.github.io/2018/10/14/Ptrace-Linux%E4%B8%AD%E4%B8%80%E7%A7%8D%E4%BB%A3%E7%A0%81%E6%B3%A8%E5%85%A5%E6%8A%80%E6%9C%AF%E7%9A%84%E5%BA%94%E7%94%A8/)
[linux ptrace I - mmmmar - 博客园](https://www.cnblogs.com/mmmmar/p/6040325.html)
[linux中fork（）函数详解 - 学习记录园 - 博客园](https://www.cnblogs.com/dongguolei/p/8086346.html)
[[译] 玩转ptrace (一) - twoon - 博客园](https://www.cnblogs.com/catch/p/3476280.html)
[打印进程内存信息 | 100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/print-process-memory.html)
[pwn从入门到放弃第三章——gdb的基本使用教程 | PWN? PWN!](https://ch4r1l3.github.io/2018/06/22/pwn%E4%BB%8E%E5%85%A5%E9%97%A8%E5%88%B0%E6%94%BE%E5%BC%83%E7%AC%AC%E4%B8%89%E7%AB%A0%E2%80%94%E2%80%94gdb%E7%9A%84%E5%9F%BA%E6%9C%AC%E4%BD%BF%E7%94%A8%E6%95%99%E7%A8%8B/)
[c - objdump -d输出汇编的含义 - Thinbug](https://www.thinbug.com/q/21742827)
[c - 为什么GDB和Objdump的指令地址相同？ - Thinbug](https://www.thinbug.com/q/50594438)
[GDB调试 - devbins blog](https://devbins.github.io/post/gdb/)
[调试器工作原理之二——实现断点(ptrace) - CodeAntenna](https://codeantenna.com/a/PsCThgFbhG)
[汇编语言基础:寄存器和系统调用 - Yungyu - 博客园](https://www.cnblogs.com/yungyu16/p/13024485.html)
[在程序地址上打断点 | 100个gdb小技巧](https://wizardforcel.gitbooks.io/100-gdb-tips/content/break-on-address.html)
[C语言指针转换为intptr_t类型 - Rabbit_Dale - 博客园](https://www.cnblogs.com/Anker/p/3438480.html)
[x86_64汇编基础 - ym65536 - 博客园](https://www.cnblogs.com/ym65536/p/4542646.html)
[GDB调试-从入门实践到原理](https://mp.weixin.qq.com/s/XxPIfrQ3E0GR88UsmQNggg)
[objdump(Linux)反汇编命令使用指南_wang.wenchao的博客-CSDN博客_linux 反汇编](https://blog.csdn.net/wwchao2012/article/details/79980514)
[解决munmap_chunk(): invalid pointer和Segmentation fault的bug_summer的专栏-CSDN博客_munmap_chunk()](https://blog.csdn.net/xiamo20149/article/details/51964945)
[调试器工作原理之二——实现断点(ptrace)_weixin_33895016的博客-CSDN博客](https://blog.csdn.net/weixin_33895016/article/details/92754025)
[gdb查看当前汇编指令_counsellor的专栏-CSDN博客_gdb 查看汇编](https://blog.csdn.net/counsellor/article/details/100034080)
[objdump命令的使用_北落师门'的专栏-CSDN博客_objdump](https://blog.csdn.net/beyondioi/article/details/7796414)
[进程与进程描述符（task_struct）_Steve_Abelieve-CSDN博客_进程描述符](https://blog.csdn.net/lizhidefengzi/article/details/70231334)
[Linux 查看进程内存分布_谈谈1974-CSDN博客_linux 查看内存布局](https://blog.csdn.net/weixin_45505313/article/details/105287599)
[Linux下关闭ALSR(地址空间随机化)的方法_counsellor的专栏-CSDN博客_linux关闭地址随机化](https://blog.csdn.net/counsellor/article/details/81543197)
[Linux平台的ASLR机制_加号减减号的博客-CSDN博客_aslr linux](https://blog.csdn.net/Plus_RE/article/details/79199772)
[ELF entry point和装载地址_ayu_ag的专栏-CSDN博客_elf入口地址](https://blog.csdn.net/ayu_ag/article/details/50737209)
[linux中如何断点调试程序,开发一个Linux调试器（二）：断点_刘为龙的博客-CSDN博客](https://blog.csdn.net/weixin_34567845/article/details/116691038)
[Linux 命令（1）—— nm 命令_baboon_chen-CSDN博客](https://blog.csdn.net/I_just_smile/article/details/106174289)
[Linux X86_64位虚拟地址空间布局与试验_Meows的牧场-CSDN博客_虚拟空间64位](https://blog.csdn.net/Wu_Roc/article/details/77203480)
[深入理解 Linux 位置无关代码 PIC_内核工匠-CSDN博客](https://blog.csdn.net/feelabclihu/article/details/108289461)
[Linux ELF装载过程及64位地址空间布局_@HDS的博客-CSDN博客](https://blog.csdn.net/weixin_44395686/article/details/104761488)
[linux内存地址断点,开发一个 Linux 调试器（三）：寄存器和内存_别总叫我大叔的博客-CSDN博客](https://blog.csdn.net/weixin_42410566/article/details/116819824)
[链接与加载-NJU-JYY_Adenialzz的博客-CSDN博客](https://blog.csdn.net/weixin_44966641/article/details/120616894)
[自己写调试器 软断点 [Linux]_氺在飛天-CSDN博客](https://blog.csdn.net/neilhhw/article/details/14408637)
[linux调试器的实现---断点的实现_darmao的博客-CSDN博客_linux打断点](https://blog.csdn.net/darmao/article/details/78535329)
[LINUX 逻辑地址、线性地址、虚拟地址和物理地址_十一月zz的博客-CSDN博客_linux 逻辑地址](https://blog.csdn.net/baidu_35679960/article/details/80463445)
[详解：物理地址，虚拟地址，内存管理，逻辑地址之间的关系_17岁boy的博客-CSDN博客_逻辑地址与物理地址](https://jrhar.blog.csdn.net/article/details/78508795)
[调试器工作原理系列三篇_关注 Linux c/c++ 数据存储 网络 算法......-CSDN博客](https://blog.csdn.net/syzcch/article/details/8350189)
[Linux源码分析之Ptrace_七月冷雨-CSDN博客_linux ptrace](https://blog.csdn.net/u012417380/article/details/60468697)
[x86_64汇编之六：系统调用（system call）_ponnylv的博客-CSDN博客](https://blog.csdn.net/qq_29328443/article/details/107250889)
[Linux Ptrace 详解_七月冷雨-CSDN博客_linux ptrace](https://blog.csdn.net/u012417380/article/details/60470075)
[玩转ptrace(二)_zhangmiaoping23的专栏-CSDN博客](https://blog.csdn.net/zhangmiaoping23/article/details/51405281)
[linux系统：ptrace系统调用浅析_老王不让用的博客-CSDN博客_ptrace 函数调用](https://wangquan.blog.csdn.net/article/details/108471212)
