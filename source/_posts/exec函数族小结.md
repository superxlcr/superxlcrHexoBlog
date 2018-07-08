---
title: exec函数族小结
tags: [操作系统,linux,cpp]
categories: [操作系统]
date: 2017-05-13 17:27:40
description: exec函数族说明、exec函数族使用情况、exec函数族语法介绍、exec函数族调用举例
---
本人最近了解了关于exec函数族相关的知识，在此进行一下总结。
# exec函数族说明
fork函数是用于创建一个子进程，该子进程几乎是父进程的副本。而当我们希望子进程去执行另外的程序时，exec函数族就提供了一个在进程中启动另一个程序执行的方法。它可以根据指定的文件名或目录名找到可执行文件（这里的可执行文件既可以是二进制文件，也可以是Linux下任何可执行脚本文件），并用它来取代原调用进程的数据段、代码段和堆栈段，在执行完之后，**原调用进程的内容除了进程号外，其他全部被新程序的内容替换了**。

# exec函数族使用情况
一般而言，在Linux中使用exec函数族主要有以下两种情况：

1. 进程认为自己不能再为系统和用户做出任何贡献时，就可以调用任何exec 函数族让自己重生
2. 一个进程想执行另一个程序，它可以调用fork函数新建一个进程，然后调用任何一个exec函数使子进程重生

# exec函数族语法介绍
exec函数族中并没有exec函数，但有6个以exec开头的成员函数，它们所需的头文件均为&lt;unistd.h&gt;，原型如下：

```cpp
int execl(const char *path, const char *arg, ...)
int execv(const char *path, char *const argv[])
int execle(const char *path, const char *arg, ..., char *const envp[])
int execve(const char *path, char *const argv[], char *const envp[])
int execlp(const char *file, const char *arg, ...)
int execvp(const char *file, char *const argv[])
```


函数的返回值为int类型，一般而言有两种情况：

- 当文件执行成功时，函数不会返回任何东西
- 当文件执行失败时，函数会返回-1，并且失败原因会记录在errno中（&lt;errno.h&gt;头文件中，一个表示错误类型的int）

这六个函数在函数名和使用语法的规则上都有细微的区别，下面就可执行文件查找方式、参数表传递方式及环境变量这三个方面进行比较说明。




## 查找方式
exec函数族中前四个函数（execl、execv、execle、execve）的查找方式都是完整的文件目录路径，而最后2个函数（也就是以p结尾的两个函数）可以只给出文件名，系统就会自动从环境变量“$PATH”所指出的路径中进行查找


## 参数传递方式
exec函数族的参数传递有两种方式，以函数名的第5位字母来区分：


1. 字母为“l”（list）的表示逐个列举的方式，最后一个指针要求是NULL
2. 字母为“v”（vector）的表示将所有参数整体构造成指针数组传递，然后将该数组的首地址当做参数传给它，数组中的最后一个指针也要求是NULL

## 环境变量

exec函数族使用了系统默认的环境变量，也可以传入指定的环境变量。这里以“e”（environment）结尾的两个函数execle、execve就可以在envp[]中指定当前进程所使用的环境变量替换掉该进程继承的所有环境变量。



总的来说，exec函数族命名关系可以归结为：
前四位：统一为exec
第五位：

- l：表示通过逐个列举的方式传入参数（execl、execle、execlp）
- v：表示通过构造指针数组的方式传入参数（execv、execve、execvp）

第六位：


- e：可传递新进程环境变量（execle、execve）
- p：可执行文件查找方式为文件名（execlp、execvp）

# exec函数族调用举例


```cpp
// 参数指针数组，以NULL结尾
char *const ps_argv[] ={"ps", "-o", "pid,ppid,pgrp,session,tpgid,comm", NULL};
// 环境变量
char *const ps_envp[] ={"PATH=/bin:/usr/bin", "TERM=console", NULL};

// 使用文件路径查找
execl("/bin/ps", "ps", "-o", "pid,ppid,pgrp,session,tpgid,comm", NULL);
execv("/bin/ps", ps_argv);

// 使用文件路径查找，指定新的环境变量
execle("/bin/ps", "ps", "-o", "pid,ppid,pgrp,session,tpgid,comm", NULL, ps_envp);
execve("/bin/ps", ps_argv, ps_envp);

// 使用文件名查找
execlp("ps", "ps", "-o", "pid,ppid,pgrp,session,tpgid,comm", NULL);
execvp("ps", ps_argv);
```




