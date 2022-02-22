---
title: 【Linux】Linux实现一个简易shell
date: 2022-2-22
tags: [Linux]
cover: https://www.helloimg.com/images/2022/02/22/GraTAc.webp
mathjax: true
---

## 1. fork()函数

​        当程序调用fork()函数并返回成功之后，程序就将变成两个进程，调用fork()者为父进程，后来生成者为子进程。这两个进程将执行相同的程序文本，但却各自拥有不同的栈段、数据段以及堆栈拷贝。子进程的栈、数据以及栈段开始时是父进程内存相应各部分的完全拷贝，因此它们互不影响。

fork()函数在Linux中有两次返回，在父进程中返回子进程的pid，在子进程中返回0。

```cpp
#include <unistd.h>
int main(void){
    pid_t pid;
    //调用一次，返回两次，在父进程中返回子进程的pid在子进程中返回0
    pid = fork();
    if(pid>0){
        printf("I'm a parent\n");
    }else if(pid==0){
        printf("I'm a child\n");
    }else{
        perror("fork");
    }
}
```

输出

```
I'm a parent
I'm a child
```

## 2. 进程等待之wait() & waitpid()

+ 如果子进程已经退出，调用wait/waitpid会立即返回，并且释放资源，获得子进程退出信息
+ 如果在任意时刻调用wait/waitpid，子进程存在且正常运行，则父进程可能阻塞
+ 如果不存在该子进程，则立即出错返回
+ 子进程的退出是个异步事件（子进程可以在父进程运行的任何时刻终止）

### 2.1 wait()

```cpp
头文件：#include<sys/wait.h>
       #include<sys/type.h>
原型
pid_t wait(int *status)
返回值：
    成功：返回被等待进程（子进程）pid
    失败：返回-1
参数：输出型参数，获取子进程退出状态，不关心则可以设置为NULL
```

### 2.2 waitpid()

```cpp
头文件： #include<sys/type.h>
        #include<sys/wait.h>
返回值：
   (1)当正常返回的时候waitpid返回收集到的子进程的进程ID

   (2)如果设置了选项WNOHANG，而调用中waitpid发现已经没
      有已经可以退出的子进程可收集，则返回0

   (3)如果调用中出错，则返回-1，这时errno会被设置成相应的
      值以指示错误所在
原型：
   pid_t waitpid(pid_t pid,int *status,int options)

    1.pid
      pid=-1，等待任一个子进程，与wait等效
      pid>0,等待其进程ID与pid相等的子进程
    2.status
       WIFEXITED(status):    若为正常终止子进程返回的状态，则为
                             真（此参数是查看进程是否是正常退出）

       WEXITSTATUS(status):  若WEXITSTATUS非零，提取子进程退
                             出码（查看进程的退出码）
    3.options
         WNOHANG：若pid指定的子进程没有结束，则waitpid()函数
                  返回0，不予以等待，若正常结束，则返回该子进程的ID
```



## 3. 进程替换：exec 函数族
所谓exec函数族，其实有六种以exec开头的函数，统称exec函数：execl、execlp、execle、execv、execvp、execve。当进程调用一种exec函数时，该进程的用户空间代码和数据完全被新程序替换，从新程序的启动例程开始执行。调用exec并不创建新进程，所以调用exec前后该进程的id并未改变。将当前进程的.text、.data替换为所要加载的程序的.text、.data，然后让进程从新的.text第一条指令开始执行，但进程ID不变。

### 3.1 exec函数族一般规律：

exec函数一旦调用成功即执行新的程序，不返回。只有失败才返回，错误值-1。所以通常我们直接在exec函数调用后直接调用perror()和exit()，无需if判断。

exec函数族名字很相近，使用起来也很相近，它们的一般规律如下：

l (list)                           命令行参数列表

p (path)                       搜素file时使用path变量

v (vector)                    使用命令行参数数组

e (environment)       使用环境变量数组,不使用进程原有的环境变量，设置新加载程序运行的环境变量

### 3.2 带p的exec函数

这类函数有：execlp，execvp

具体说明：表示第一个参数无需给出具体的路径，只需给出函数名即可，系统会在PATH环境变量中寻找所对应的程序，如果没找到的话返回－1。

```cpp
int execvp(const char *file, char *const argv[]);
```

execvp()会从PATH 环境变量所指的目录中查找符合参数file 的文件名，找到后便执行该文件，然后将第二个参数argv传给该欲执行的文件。

## 4. 代码实现和结果

github链接：[GitHub - Kakaluoto/MyShell](https://github.com/Kakaluoto/MyShell)

```cpp
#include <iostream>
#include <vector>
#include <cstring>
#include <unistd.h>
#include <sys/wait.h>

#define cd_failed 0
#define cd_success 1
using namespace std;

vector<string> argv;//存储当前命令所有参数
vector<string> history_cmds(100);//存放历史命令，因为时间有限功能还未实现
string cmd;//当前命令字符串
char* current_path = nullptr;//当前工作路径

void argparse(); //解析参数
void change_directory();//cd命令
void execute_cmd();//执行命令


int main() {
    current_path = getcwd(NULL, 0);//获取当前路径
    while (1) {
        // 前置输出提示这是一个shell
        printf("myshell:%s$ ", current_path);
        getline(cin, cmd);
        // 如果输入为exit 则结束当前进程
        if (strcmp(cmd.data(), "exit") == 0) {
            delete current_path;
            return 0;
        }
        argparse();
        execute_cmd();
        argv.clear();
        // 后面在do_cmd部分会解释为什么无循环结束条件
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

int change_directory(int argc) {//cd命令
    if (argc == 2) {
        if (chdir(argv[1].data()) == 0) {//成功返回0，失败返回-1
            current_path = getcwd(NULL, 0);
            if (current_path != nullptr) {
                return cd_success;
            } else {
                cout << "No such file or directory!\n";
                return cd_failed;
            }
        } else {
            cout << "No such file or directory!\n";
            return cd_failed;
        }
    } else {
        cout << "too many arguments!\n";
        return cd_failed;
    }
}

void execute_cmd() {
    pid_t pid;
    int argc = argv.size();
    char** arguments = new char* [argc];//转换参数类型，以便能够喂到exec函数
    for (int i = 0; i < argc; i++) {
        arguments[i] = (char*) argv[i].data();
    }
    if (strcmp(arguments[0], "cd") == 0) {
        change_directory(argc);//执行cd命令
    } else {
        switch (pid = fork()) {
            //fork子进程失败
            case -1:
                cout << "Failed to create subprocess!\n";
                return;
                //处理子进程
            case 0:
                execvp(arguments[0], arguments);
                //子进程，没有成功执行
                cout << "invalid input command : \"" << arguments[0] << "\"" << endl;
                exit(1);
            default: {
                int status;
                waitpid(pid, &status, 0);//等待子进程返回
                int err = WEXITSTATUS(status); // 读取子进程的返回码
                if (err)cout << "Error: " << strerror(err) << endl;
            }
        }
    }
}
```



进入MyShell可执行文件所在目录执行如下命令即可

```sh
./MyShell
```

得到输出如下

```sh
myshell:/home/hy/myCppProject/cmake_demo/myshell$ 
```

执行ls

```
myshell:/home/hy/myCppProject/cmake_demo/myshell$ ls
MyShell  MyShell.cpp  readme.md
```

执行ls -l：

```sh
myshell:/home/hy/myCppProject/cmake_demo/myshell$ ls -l
total 176
-rwxrwxr-x 1 hy hy 159632 12月 18 23:51 MyShell
-rw-rw-r-- 1 hy hy   3020 12月 19 16:08 MyShell.cpp
-rw-rw-r-- 1 hy hy  13420 12月  9 22:43 readme.md

```

执行ps

```sh
myshell:/home/hy/myCppProject/cmake_demo/myshell$ ps
    PID TTY          TIME CMD
 135724 pts/0    00:00:00 bash
 135766 pts/0    00:00:00 MyShell
 135847 pts/0    00:00:00 ps
```

执行cd：

```sh
myshell:/home/hy/myCppProject/cmake_demo/myshell$ cd /
myshell:/$ 
```

执行pwd和ls命令

```sh
myshell:/$ pwd
/
myshell:/$ ls
bin   cdrom  etc   lib	  lib64   media  opt   root  sbin  srv	     sys  usr
boot  dev    home  lib32  libx32  mnt	 proc  run   snap  swapfile  tmp  var
```



