## 16 关于I/O流分离的其他内容

### 16.1 分离I/O流

#### 16.1.1 2次I/O流分离

之前通过两种方法分离了I/O流

- 第一种是第10章的"TCP I/O过程分离"。这种方法通过调用fork函数复制出1个文件描述符，以区分输入和输出中使用的文件描述符。分开了2个文件描述符的用途

- 第二种是第15章 通过2次fdopen函数的调用，创建读模式FILE指针和写模式FILE指针。分离了输入工具和输出工具

#### 16.1.2 分离流的好处

第十章的"流"分离目的。
- 通过分开输入过程(代码)和输出过程降低实现难度
- 与输入无关的输出程序可以提高速度

第15章的"流"分离目的
- 为了将FILE指针按读模式和写模式加以区分
- 可以通过区分读写模式降低实现难度
- 通过区分I/O缓冲提高缓冲性能

#### 16.1.3 流分离带来的EOF问题

第7章介绍EOF的传递方法和半关闭的必要性。
> shutdown(sock, SHUT_WR)

第10章的流分离没有问题，但是15章的基于fdopen函数的流则不同，我们不知道在这种情况下如何进行半关闭，我们先尝试针对输出模式的FILE指针调用fclose函数这种错误的方法

[sep_clnt.c](./sep_clnt.c)

[sep_serv.c](./sep_serv.c)

```
gcc sep_clnt.c -o clnt
gcc sep_serv.c -o serv
./serv 9190
./clnt 127.0.0.1 9190
```

![](https://s2.ax1x.com/2019/02/24/k4WdPI.png)

运行结果可知服务器端未能接收最后的字符串

### 16.2  文件描述符的复制和半关闭

#### 16.2.1 终止流时无法半关闭的原因

![](https://camo.githubusercontent.com/2d82e3d9cc978724b5c1fc7409fe4242072b4759/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30312f33302f356335313231646138393935352e706e67)

可以看出，上图中读模式FILE指针和写模式FILE指针都是基于同一文件描述符创建的。因此针对任意一个FILE指针调用fclose函数都会关闭文件描述符，也就是终止套接字

![](https://camo.githubusercontent.com/994831b36e384970cee4762e838279133d961b1d/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30312f33302f356335313232343035313830322e706e67)

那么如何进入可以输入但无法输出的半关闭状态呢？ 其实只需要创建FILE指针前先复制文件描述符即可

![](https://camo.githubusercontent.com/18d36df8381855e1998235ebce5273cde2d2d540/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30312f33302f356335313232613435633566312e706e67)

因为**销毁所有文件描述符候才能销毁套接字**，所以说针对写模式FILE指针调用fclose函数后，只能销毁与该FILE指针相关的文件描述符而不能销毁套接字

![](https://camo.githubusercontent.com/a45757055634ba0e57d91beb392be9e35ebc3936/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30312f33302f356335313233616437646633312e706e67)

上图是否已经完成半关闭？ **并没有** 因为还剩一个文件描述符，而且该文件描述符也可以同时进行I/O ， 因此没有发送EOF，仍可以通过该文件描述符进行输出


#### 16.2.2 复制文件描述符

显然不会是通过fork函数，因为fork函数将复制整个进程。而我们想要的是在同一进程内完成描述符的复制

![](https://camo.githubusercontent.com/f4644cbc98b93685f77a6d0b7b38bfc79e859440/68747470733a2f2f692e6c6f6c692e6e65742f323031392f30312f33302f356335313235373963343562362e706e67)

复制完成后，两个文件描述符都可以访问文件，但是两者的值不同


#### 16.2.3 dup&dup2

通过下面两个函数都可以完成对文件描述符的复制
```c++
#include <unistd.h>

int dup(int fildes);

int dup2(int fildes,int fildes2);
/*
fildes  : 需要复制的文件描述符
fildes2 : 明确指定的文件描述符整数值
*/
```
dup2 函数明确指定复制的文件描述符的整数值。向其传递大于 0 且小于进程能生成的最大文件描述符值时，该值将成为复制出的文件描述符值。下面是代码示例：

[dup.c](./dup.c)

```
gcc dup.c -o dup
./dup
```

![](https://s2.ax1x.com/2019/02/24/k45KeO.png)

#### 16.2.4 复制文件描述符后流的分离

[sep_serv2.c](./sep_serv2.c)

[sep_clnt.c](./sep_clnt.c)

![](https://s2.ax1x.com/2019/02/24/k47q8H.png)

运行结果证明服务器在半关闭状态下向客户端发送了EOF

> 无论复制出多少文件文件描述符，均应调用shutdown函数发送EOF并进入半关闭状态。

### 16.3 习题

> 以下答案仅代表本人个人观点，可能不是正确答案。

1. **下列关于 FILE 结构体指针和文件描述符的说法正确的是？**

答: 加粗代表正确

- 与 FILE 结构体指针相同，文件描述符也分输入描述符和输出描述符
- 复制文件描述符时将生成相同值的描述符，可以通过这 2 个描述符进行 I/O
- 可以利用创建套接字时返回的文件描述符进行 I/O ，也可以不通过文件描述符，直接通过 FILE 结构体指针完成
- **可以从文件描述符生成 FILE 结构体指针，而且可以利用这种 FILE 结构体指针进行套接字 I/O**
- 若文件描述符为读模式，则基于该描述符生成的 FILE 结构体指针同样是读模式；若文件描述符为写模式，则基于该描述符生成的 FILE 结构体指针同样是写模式

2. EOF 的发送相关描述中正确的是？

答: 加粗代表正确

- **终止文件描述符时发送 EOF**
- **即使未完成终止文件描述符，关闭输出流时也会发送 EOF**
- 如果复制文件描述符，则包括复制的文件描述符在内，所有文件描述符都终止时才会发送 EOF
- **即使复制文件描述符，也可以通过调用 shutdown 函数进入半关闭状态并发送 EOF**