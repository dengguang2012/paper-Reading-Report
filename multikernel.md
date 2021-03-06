---
layout: post
title: multikernel
categories:
- paper
tags:
- technology
- 
---

##multikernel阅读报告

邓志会 2015210926

元东 2015210938

multikernel: 对可扩展多核系统的新的操作系统架构

原文相关信息

The Multikernel: A new OS architecture for scalable multicore systems

Andrew Baumann, Paul Barhamy, Pierre-Evariste Dagandz, Tim Harrisy, Rebecca Isaacsy,
Simon Peter, Timothy Roscoe, Adrian Schüpbach, and Akhilesh Singhania

Systems Group, ETH Zurich

Microsoft Research, Cambridge zENS Cachan Bretagne


商业计算机系统包含越来越多的处理器内核并且展示了越来越多的不同的架构权衡，包括内存层次，互联，指令集和输入输出配置。先前的高性能计算系统已经能够在特定性狂下可扩展，但是现代客户端服务器负载动态本质，与为所有操作系统负载和硬件的负载静态优化相违背，这构成操作系统结构巨大挑战。

Barrelfish该系统由微软剑桥研究院和苏黎世理工学院联合全新开发，专为现在和未来的多核心(Multi-Core)、众核心(Many-Core)处理器环境而设计，通过在各个核心之间建立一条网络总线来从根本上提升系统效率和性能. 在硬件水平飞速发展和性能需求不断提升的同时，现有操作系统的内核架构已经无法很好地高效利用相应资源，特别是存在资源共享机制的局限。Barrelfish则通过自己的总线在处理器核心之间传递信息，并采用类似数据库的方式来跟踪可用硬件资源。Barrelfish不再通过驱动程序将应用软件与硬件设备完全隔离，而是存在一个某种数据库，其中可以找到大量有关硬件的低级信息。系统内核则是单线程和非抢占的。调度和信息传递相结合，信息到达后就直接激活等待中的线程。它还用到了一些微核(microkernel)概念，在保护空间内运行驱动程序。

作者认为多核硬件未来挑战在于最好的发挥机器网络特性，使用分布式系统的理念来重新思考操作系统架构。作者研究了一个新的操作系统架构，multikernel,它把机器当作独立内核构成的网络，假设最底层没有内核之间的共享，把传统的操作系统功能移动到一个分布式系统进程，它们通过消息传递来通信。

作者已经实现了一个multicore操作系统显示这个方法有潜力，还描述了操作系统传统的可扩展问题（比如内存管理）可以使用消息机制有效解决，可以利用对分布式系统和网络的深刻理解。在多核系统上原型评测表明即使在现在的机器上，multicore的性能可以和传统的操作系统相比较，并且能更好的扩展支持未来硬件。


### 背景介绍

计算机硬件比系统软件变化更快。内核，caches，互联链接不同的混合，输入输出设备和加速器，结合不断增加的内核数量，导致了大量的扩展和对于操作系统设计者正确性的挑战。

作者使用锁保护的数据结构共享内存内核的基本结构解决工程问题。作者对于操作系统结构作为分布式系统函数单元进行了重新思考，通过明确的消息进行通信。作者指出了三个设计原则：使得所有的内核之间通信更加明确；使得操作系统结构成为中立硬件；视状态为复制而不是共享。
 
 <img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/32.png" style="width: 50%; height: 50%"/>​

 
作者开发的模型称为multicore,不仅更好匹配了底层硬件(网络，异构和动态)，但这使得作者可以应用从分布式系统到问题规模的理解，为未来硬件操作系统多样性。

即使现有系统高效缓存一致性共享内存，使用基于消息而不是共享数据通信构建一个操作系统提供有形的益处，而不是顺序操纵共享数据结构，这被远程的数据访问所限制，管道化能力和批量化消息编码远程操作使得单核能够达到更大的吞吐量并且减少互联使用，此外，这个概念自然地容纳异构硬件。

这项工作贡献在于：作者介绍了多核模型和明确通信的设计原则，硬件中立结构和状态复制。

作者展示了multikernel,Barrelfish,这探索启示应用模型到具体的操作系统实现。

作者通过测量Barrelfish满足可扩展和适应硬件特性的目标，并且在同时代的硬件上提供了竞争性能。

### motivation

今天构建的大多数的计算机有多核处理器，未来内核数量还将会增加。然而，商业多核服务器已经在单一的操作系统镜像上扩展到数百处理器，处理tb量级的ram和多个10gb网络连接。作者认为是否为未来的多核硬件需要新的操作系统技术，还是商业操作系统只需使用已有的技术在大的多核处理器系统。在这个部分，作者认为，面对未来的商品硬件通用操作系统的一个挑战是不同于那些用于传统ccNUMA和SMP机器有关的高性能计算，也为通用系统软件奠定额外的可扩展性的挑战。系统日益多样化，一个通用的操作系统今天在一个日益多样化的系统设计必须表现良好，每一个有不同的性能特点。这意味着，与高性能计算的大系统不同，这样的操作系统不能在任何特定的硬件配置的设计或执行时间进行优化。内核日益多样化，多元化不仅仅在商品机器的范围是一个挑战，在一台机器内，内核可以很不同，而这一趋势是向着一个混合的不同的内核。有些会有相同的指令集架构（ISA）但不同的性能特点，因为一个大的核心处理器将不容易并行化的程序，但只使用一个小的内核会上表现不佳在一个程序连续的部分。其他核具有特殊功能不同的指令集，许多外围设备（GPUs、网络接口等，通常的基于FPGA处理器的可编程专用处理器越来越互联越来越重要。目前的操作系统设计中的通用内核之间的区别假定为是均匀的，并运行一个单一的，共享的内核，和外围设备通过一个狭窄的驱动程序接口的实例访问。然而，作者没有看到操作系统管理软件运行在多CPU核心管理。此外，核心的异质性意味着核心可以不再共享一个单一的操作系统内核的实例，也是因为性能的权衡各不相同，或因为指令集仅仅的不同。

即使对于现代的高速缓存相干的多处理器，消息传递的硬件已经取代了单一的共享互联因为可扩展性原因。硬件因此类似于消息传递网络，如在图2中的商品PC服务器的互连拓扑结构。而在最新的硬件cache 一致性协议之间的CPU保证了操作系统能继续安全地假定一个单一的共享内存，如路由和拥塞的网络问题，在大型多处理器的著名问题，现在问题在商品机器互连问题。

消息开销少于共享内存。

后期，共享内存系统一直是最适合的电脑硬件的性能和良好的软件工程，但这一趋势正在逆转。作者可以看到这样的证据，通过一个实验，比较使用共享内存更新数据结构的成本和使用消息传递的成本。为各种大小的4×4核AMD系统更新对内核数量延迟。
cache一致也不是万能的，由于内核的数目和随后的互连的复杂性，硬件cache缓存一致性协议将变得开销越来越大。其结果是，它是一个独特的可能性，未来的操作系统将不得不处理非相干内存，或将能够绕过高速缓存一致性协议实现大幅性能收益。

###消息变得更简单

有合法的软件工程议题与基于消息的软件系统有关，和一个可能因此提问的智慧基于“无共享”模型构建一个多处理器操作系统，有2个主要的问题，第一个是不能够访问共享数据，其次是异步消息导致的与事件驱动的编程风格。

然而，共享数据的方便性有些肤浅。还有当使用共享数据结构的正确性和性能的缺陷，和可扩展的共享内存的程序（尤其是高性能科学计算应用），专业开发者在锁粒度的细节上面非常仔细。通过在一个低的水平微调代码，可以最大限度地减少所需的缓存的共享数据，并减少争用高速缓存的所有权。这减少了当缓存内容过期时所产生的互连带宽和处理器的数目停顿。

###多核模型

在这部分作者提出对于异构多核机器的操作系统架构，作者称之为多核模型。概括地说，作者将系统作为一个分布式系统的核心，使用消息和不共享内存来通信，多核模型的三个设计指导原则如下：

	1 使得所有内核之间的通信明确。
	2 使得操作系统结构是硬件独立的。
	3 把状态视为重复而不是共享的。

这些设计原则使得操作系统从分布式系统方法获益，能获得改善性能，自然地支持硬件异构，更好的模块化，和重利用分布式系统已经开发算法的能力。

使得内核之间通信明确

在多核操作系统，所有的内核之间的通信是使用显式报文进行。推论的结果是，在每个内核上运行的代码之间没有内存，除了用于消息传递的通道。正如作者所看到的，使用消息访问或迅速更新状态变得比共享内存访问的缓存行的数目增加更有效的。作者可以期待这样的效果在未来变得更加明显。请注意，这并不妨碍应用程序之间共享内存，只有这样的操作系统设计本身不依赖于它。

明确的通信模式，方便推理系统互连的使用。对比隐式通信（如分布式共享内存，或用于高速缓存一致性的消息），共享状态的知识，何时能被访问，被谁释放这个权限。当然，在操作系统内核中建立的实践，设计点解决方案数据结构，可以使用只有1个或2个缓存未命中特定的架构更新，但它是很难发展这样的操作硬件变化，这样的优化忽略了更广泛的图片涉及多个结构操作系统的更大的状态更新的。
作者以前认为，随着机器越来越像一个网络，操作系统将不可避免地作为一个分布式系统。显式通信允许操作系统部署知名的网络优化，使得能够更有效地利用互连，如流水线，和批处理。在第5.2节中，作者展示了在分布式的能力管理的情况下这种技术的好处。

这种方法也使操作系统提供隔离和异构内核上的资源管理，或安排工作，有效地对内核之间的拓扑结构，将任务置于参考通信模式和网络的影响。此外，消息的抽象对于生成内核是缓存不一致，或者不共享内存。
消息传递允许操作可能需要分裂通信阶段，作者的意思是，该操作发送请求，并立即继续，期望一个答复将在未来一段时间内到达。当请求和响应是分离的，内核发出请求可以做有用的工作，或睡眠来节省电力，同时等待答复。一个共同具体的例子是远程缓存失效。在一个高度并发的情况下，提供了不需要完成无效的正确性，它可以更重要的是不浪费时间等待操作完成，比执行它与最小的可能的延迟。

最后，基于明确通信的一个系统是适合于人类或自动化分析。一个消息传递系统的结构自然是模块化的，因为只有通过定义明确的接口组件进行通信。因此，它可以更容易地进化和更容易地细化和对故障的鲁棒性。

使得操作系统是硬件中立的

multikernel尽可能的分隔操作系统结构和硬件。这意味着只有两类操作系统作为整体目标是特定的机器架构-消息传输机制，硬件接口(CPUs和设备)。这有效面的几个好处：首先，适应操作系统运行在硬件上有新的性能特点将不需要扩展，交叉改变到基础代码。当部署的系统越来越多样化这也越来越重要。
特别是，经验表明进程之间通信机制关键在于指定硬件的优化。多核内的独立硬件意味着可以从硬件实现细节上分隔分布式通信算法。

作者设想一些不同的消息传递的实现（例如，一个使用共享内存，用户级RPC协议或基于硬件的可编程外围通道。硬件平台不存在缓存一致性，甚至没有共享内存，很可能会变得更为广泛。一旦消息传输进行了优化，作者可以独立的实现高效的基于消息的算法，而与硬件细节或内存布局独立。

最后的优点是使协议实现和消息传输的后期绑定。例如，不同的传输可以用于在输入输出链路上的核心，或者通过调整队列长度或轮询频率来调整所观察到的工作量。在5.1节中作者展示了一个拓扑感知组播报文协议可以对商品操作系统超越高度优化的TLB强行终止机制。

视状态为可重复的
操作系统保持状态，其中一些，比如Windows调度员数据库或者Linux调度队列，一定是多个处理器可以访问的。传统的那个状态存在于被锁保护的共享数据结构，然而，在一个多核中，明确多核之间的计算不共享内存自然地导致一个内核之间全局操作系统复制状态模型。

复制用于操作系统的可扩展性是一个众所周知的技术，但一般是作为一个优化实现，否则共享内存内核设计。相反，任何在多核潜在的共享状态的访问就好像它是一个本地副本更新。在一般情况下，该状态是复制的，尽可能多的是有用的，并通过交换消息保持一致性。根据所需的一致性语义，更新可以是长期运行的操作，以协调副本，并因此被暴露在非阻塞和分裂阶段。作者提供了不同的更新语义的例子。

复制的数据结构可以提高系统的可扩展性，通过减少系统互连负载，争用内存，和同步的开销。使数据更接近核心的过程，这将导致访问延迟降低。

复制支持领域不共享内存，无论是未来的通用设计或现今的可编程外设的领域，是必需的，特别是专业数据结构的内核设计。使多核设计内在的状态复制更容易保存的操作系统结构和算法作为底层硬件的发展。

此外，复制是一个有用的框架，在一个操作系统设定的运行核心，或者当hotplugging处理器，或者当关闭硬件子系统以节省电力。作者可以应用标准的结果从分布式系统的文献，以保持系统的状态的一致性，最后，对多核模型的一个重要的潜在的优化是私下分享一组紧密耦合的内核或硬件线程，通过一个共享内存同步技术像自旋锁保护。

###实现

在Barrelfish还是在多核设计空间的一个点，它还不是唯一的方式来构建多核。这个部分作者对实现进行了描述，记录了设计中哪一种选择从模型中被继承，哪些由其他的原因所激励，比如本地性能，工程缓解，自由政策等等。

 <img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/33.png" style="width: 50%; height: 50%"/>​

 Barrelfish 现在运行在基于x86-64的多核处理器上面。这个8核Intel系统有一个Intel的s5000XVN主板带有2个4核 2.66GHz Xeon X5355处理器和一个单独的外部内存控制器。每个处理器包包含2个dies,每个包含2个内核和一个共享的4MB L2高速缓存。两个处理器通过一个共享的前端总线连接到内存控制器，然而内存控制器实现了一个snoop过滤器来减少跨越总线的相关通信。

multikernel模型需要多个独立的操作系统实例通过显式报文通信。在Barrelfish作者对每个内核OS实例进入特权模式CPU驱动和一个特权级用户模式监控过程，如图5。处理器的驱动程序是纯粹的本地到一个内核，分布式系统的监视和相关的CPU驱动封装已经发现的功能在一个典型的单片微内核中：调度，通信，和低级别的资源分配。对Barrelfish剩下的部分由设备驱动程序和系统服务（如网络栈，内存分配器，等等），它在内核运行在用户级进程。设备中断路由到适当的核心硬件，解复用，内核的CPU驱动，并作为一个消息传递到驱动程序。

CPU驱动执行保护，授权、时间片的进程，并介导进入内核和相关硬件（MMU，APIC，等）。因为它与其它内核没有共享状态，CPU驱动可以完全的由事件驱动，单线程的，和不可抢占。它以用户进程或设备或其他内核的中断形式来处理事件的过程。这意味着，反过来，它比传统的内核更容易编写和调试，使其文本和数据位于内核的本地存储器。

监视器总的来说协调系统的整体状态，并封装了许多机制和政策，将在传统的操作系统内核中找到。监视器是单核的，用户空间的进程，因此可以调度的。因此，他们是非常适合的分相，面向消息的内核之间通信的多核模型，特别是在处理消息队列，和长期的远程操作。
在每一个内核上，复制数据结构，如内存分配表和地址空间映射，通过监控协议运行在全局范围内保持一致。由监视程序处理访问全局状态的应用程序请求，它可以协调访问远程副本。

多核模型导致比一个典型的单片多处理器操作系统有点不同的进程结构。在Barrefish进程代表集合的调度对象，对每一内核它可以执行。在Barrefish中通信实际上不是在进程间的而是在调度者之间的。

在一个多核中，所有的内核之间通信时和消息一起发生。作者希望使用不同的传输实现不同的硬件方案。然而，只有在当前的硬件平台唯一的内核之间通信机制，是缓存一致的。Barrelfish因此目前在内核之间使用用户级RPC的一个变种：共享内存区域作为传输信道信息的点对点之间的高速缓存行大小的单个写和读的内核。

即使一个多核操作系统它自己是分布的，它一定一贯管理一系列的全局资源，比如物理内存。特别地，因为用户级别应用程序和系统服务需要在多个内核之间利用共享内存。因为操作系统代码和数据存储在相同内存中，机器内物理内存的分配一定是一致的。例如，系统一定确保一个用户进程永远不需要获得一个虚拟映射内存区域用来存储一个硬件页表或其他的操作系统对象。

考察对扩展多核模型到用户空间应用程序编程语言运行时。然而，目前大多数的程序，Barrelfis支持传统线程过程模型在多个调度员共享一个虚拟地址空间的，通过在每个调度程序之间协调运行库。这种协调影响三OS组件：虚拟地址空间，容量，和线程管理，这是一个传统操作系统的功能可以提供一个多核的例子。一个共享的虚拟地址空间，可以通过在所有调度员之间共享硬件页表中实现，或使用协议的消息实现复制硬件页表的一致性。

至于共享地址空间，用户应用程序也期望在内核之间分享容量（例如，映射内存区域）。然而，用户的地址空间的容量，仅仅是一个内核级的数据结构的引用。该监视器提供了一种机制，在内核之间发送容量，以确保在该过程中，能力是不待撤销，并是一种可以转移类型。用户级库执行容量操作，调用监视器以保持内核之间的一致的容量空间。

###评测

评测部分对于Barrelfish好的基线性能，多核的可扩展性能，对于不同的硬件的适应性，为了性能利用消息传递抽象 和充足的模块化利用硬件拓扑意识。
对于优化当前硬件的Windows和Linux已经做了很多，相反，作者的系统是不可避免地更轻量级（它是新的，不太完整）。相反，他们应该被理解为暗示Barrelfish对当代硬件合理执行。
作者对于microbenchmark做了更强的主张。Barrelfish可以与这些行动的内核扩展的很好，并且可以很容易地适应使用更有效的沟通模式（例如，剪裁组播缓存架构和硬件拓扑结构）。最后，作者还可以演示的好处，流水线和批处理的请求消息，而不需要修改操作系统的操作系统代码。
由于Barrelfish用户环境包括标准C 数学库，虚拟内存管理，和POSIX线程子集和文件IO API，应用程序移植大多是简单的。在这个评价过程中移植了一个Web服务器，网络协议栈的，和各种驱动程序、应用程序和库，Barrelfish给了作者信心，系统设计提供了一个可行的替代现有的单片系统。然而，从无到有提出一个新的操作系统是一个实质性的承诺，并限制在作者可以充分评估多核架构的范围。特别是，这种评价不解决复杂的应用程序的工作负载，或更高级别的操作系统服务，如存储系统。此外，作者还没有评估系统在超越目前可用的商品硬件，或它的整合异构内核能力的可扩展性。

这个工作的优点在于作者对于多核机器越来越像复杂的网络系统，提出了多核架构的一种方式。操作系统对于一个分布式系统，可进行局部的优化，而不是集中的系统，必须以某种方式被缩放到现代或未来的机类网络环境。通过基于复制的数据，基于消息的核心和分裂相操作进行了操作系统设计
可能缺点在于它支持异构，但实验只在Intel和AMD 多核机器上面进行，不能够代表完全的异构环境。