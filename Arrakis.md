---
layout: post
title: Arrakis
categories:
- paper
tags:
- technology
- 
---

##Arrakis

Arrakis 控制面板操作系统阅读报告

邓志会 2015210926

元东 2015210938

最近的设备硬件趋势形成了网络服务操作系统设计的新方法。传统操作系统中，内核辅助服务器应用程序访问硬件设备，隔离了网络和磁盘安全。
作者设计实现了新的操作系统Arrakis，把内核中的传统角色分成两个。应用可以直接访问虚拟IO设备，允许大多数的IO操作完全跳过内核，
而内核不需要对每个操作进行内核调解提供网络和磁盘保护。硬件软件改变需要利用这个新的抽象，而对比调试好的Linux实现，
这个对于流行的NOSQL存储显示了2到5倍的延迟和9倍的吞吐量的提升。

### 背景介绍

减少操作系统进程抽象的开销一直是系统设计的一个长期目标。这个问题在现代客户端-服务器计算中已经特别突出。高速以太网和低延迟的持久存储的组合是为I/O密集型软件提高效率基线。
许多服务器花费大量的时间执行操作系统代码：传送中断、多路分解和复制网络数据包，并保持文件系统的元数据。服务器应用程序通常执行非常简单的功能，如键值对查找和存储，每个客户端请求都多次遍历操作系统内核。
这些趋势引起了大量针对优化内核代码路径不同用户情况下的研究，有的减少内核中冗余的复制，减少了大量连接的开销，协议专业化，资源容器，磁盘和网络缓存之间直接的传输，中断引导，
系统调用批处理，硬件TCP加速。这大量已经被主线商业操作系统所采纳，然而均走向失败，可以看到Linux网络和文件系统协议栈延迟和吞吐很多倍数的差于原始硬件。

二十年之前，研究员对于并行计算提出了流水线包处理在网络工作台上映射网络硬件直接到用户空间中。即使很多时候在商业上不成功，但是虚拟化市场已经导致硬件供应商重拾这种想法。

本文探讨了在通用服务器操作系统的背景下，从数据路径中删除内核的影响，而不是一个虚拟机监视器。作者认为，这样做必须提供的应用程序与传统的设计与相同的安全模型。
很容易通过扩展的可信计算基包括应用程序代码，获得很好的性能，允许应用程序直接访问网络/磁盘。当然高性能和安全也并不矛盾。

作者做出的三个主要贡献是：
给出一个设备硬件，内核和非特权级进程接触网络和磁盘IO操作运行时环境的分工的体系结构。而如何有效地为IO设备模拟我们的模型，不能完全支持虚拟化。

在开源Barrelfish操作系统上面实现了模型的原型，运行在商业获取的多核电脑和IO设备硬件上。

作者使用原型来量化多个广泛使用网络服务的用户层IO的潜在益处，包括分布式对象缓存，Redis,IP层次的中间件，和一个HTTP层负载均衡。显示了相对Linux的很大提升。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/11.jpg" style="width: 50%; height: 50%"/>​

###网络协议栈开销

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/12.jpg" style="width: 50%; height: 50%"/>​

网络协议栈开销：硬件中包的处理，在IP和UDP层。
调度开销：唤醒一个进程，选择运行进程和相应的上下文切换
内核交叉，从内核态到用户态和返回。
包数据的复制：从内核到用户接收的缓存和返回发送

在Linux中，共有3.36us（见表）所用处理的每一个数据包，在网络堆栈中花费近70%。这方面的工作主要是软件复用、安全检查，以及由于各个层的间接开销。内核必须验证传入的数据包的头，
并在发送数据包时进行安全检查，以提供应用程序所提供的参数。该堆栈还执行在层边界检查。调度程序的开销在很大程度上取决于接收过程是否正在运行。如果是，
在调度程序中花费的时间只有5%，如果不是，从空闲的进程到服务器进程的时间增加了一个额外的2.2 us和0.6 us的其他部分的增长。

高速缓存和多核系统的锁定争用的问题，进一步增加了开销，并加剧了事实，传入的消息可以传递到不同的队列由网络卡，导致它们被不同的处理器内核处理，
这可能不相同的用户级进程的核心，如图1所示。先进的硬件支持，如加速接收流转向旨在降低这种成本，但这些解决方案本身对平凡的安装成本

通过促使硬件支持从数据中心删除内核的调解，Arrakis可以完全消除某些类别的开销，并尽量减少其他的影响。表1也显示了两种Arrakis相应的开销。
Arrakis完全消除调度和内核交互的开销，因为数据包直接传送到用户空间。网络协议栈的处理仍然是必需的，当然，但它是大大简化了：它不再需要将不同的应用包，
和用户级网络栈不需要验证用户所提供的广泛的作为内核的实现必须的参数。因为每个应用程序都有一个单独的网络协议栈，并且数据包被传送到应用程序正在运行的内核，
锁争用和缓存效应被减少。

###存储协议栈开销

说明今天的OS存储栈的开销，我们进行一个实验，在我们立即执行小的写操作之后在紧密循环10000次迭代fsync3系统调用，测量每个操作的延迟。我们在一个RAM磁盘上存储文件系统，
所以测量的延迟代表纯粹的CPU开销。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/13.jpg" style="width: 50%; height: 50%"/>​

###应用层开销

在典型的数据中心应用程序中，这些输入/输出协议栈开销。考虑redis4 NoSQL存储。redis通过操作日志（称为附加文件）进行持续写入操作，并且可以读取内存中的数据结构。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/14.jpg" style="width: 50%; height: 50%"/>​

####短板延迟

加上平均操作延迟，大规模的部署典型涉及背端服务前后的用户请求，大量的键值对操作会在一系列服务器上面执行来服务一个用户的请求。当请求一定在等待上面这些操作完成最慢的操作决定了系统的延迟。

####硬件输入输出虚拟化

Single-Root I/O虚拟化（SR-IOV）是一个正在用于支持高速I/O的多个虚拟机共享一个单一的物理机硬件技术。SR IOV capable I/O适配器出现在PCIe互连作为一个单一的“物理功能”（PCI为设备说法），
可以动态创建额外的“虚函数”。这些虚函数是一个不同的PCI设备，可以直接映射在不同的虚拟机，并获得它可以通过IOMMU保护（例如，在英特尔的VT-d技术架构）。对于客户操作系统，
每个虚拟功能都可以被编程为一个普通的物理设备，一个正常的设备驱动程序和一个未修改的输入/输出协议栈。访问物理硬件系统管理软件创建和删除这些虚拟功能，
并配置适配器在SR-IOV适配器复用硬件操作不同的虚拟功能，因此不同的客户操作系统。

SR-IOV被用来减少操作系统虚拟化的开销，综合平台硬件支持虚拟化，它可以有效地从数据路径中去除虚拟机管理程序。然而，当特定使用时，客户端操作系统仍然调解访问数据包权限。

在Arrakis，作者使用SR-IOV的IOMMU，及配套的适配器，提供应用级直接的访问I/O设备。这是一个现代化的实现；一个特定的硬件设备驱动程序使得API相匹配在特定的硬件细节上。
当然随着现有的硬件慢慢的改进，更好地支持用户级I/ O。远程直接内存访问是另一个有名的用户层网络模型。RDMA给应用程序直接从用户空间读取或写入虚拟内存区域到远程机器上的能力，
绕过操作系统内核。

###设计和实现

为数据面操作最小化内核的参与，Arrakis设计为大多数IO操作限制或者消除内核的调制。IO请求路由到应用层的地址空间不需要内核的参与，也不需要牺牲安全和隔离性质。

对应用程序员透明：Arrakis设计用来在不需要应用写到PoSIX API 应用。

合适的操作系统/硬件抽象：Arrakis的抽象应该是足够的复杂而能够高效地支持各种IO模式，在多核系统上有很好的扩展，支持本地化和负载均衡的应用需求。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/15.jpg" style="width: 50%; height: 50%"/>​

Arrakis目标支持虚拟化的IO硬件。在这篇文章中，作者专注于可以呈现多个操作系统和在节点上运行操作系统的应用软件实例的硬件。对于每个虚拟化设备实例，潜在的物理设备提供专门的内存映射寄存器文件，描述符队列和中段，因此允许控制面板映射每个设备实例到单独的保护域。
设备输出管理接口可以通过控制面板来创建或者销毁虚拟设备实例，不需要内核干预，为了执行这些操作，应用依赖于用户层IO协议栈提供为一个库。用户层IO协议栈会是应用短板到应用程序它会有额外的访问虚拟设备实例。在这个专用的环境中不需要多路复用操作和安全检查。

###硬件模型

作者工作的一个关键点是开发虚拟化的I / o的硬件独立层、设备模型提供了一个“理想”的硬件特性集合。此设备模型捕捉到硬件实现在硬件上的数据平面操作的传统内核所需的功能。模型类似于已经被提供一些硬件的输入/输出适配器。

网络设备提供支持表现出多个虚拟网络接口卡(VNICs)的虚拟化。它们可以基于复杂的过滤表达式多路复用或者解复用包，不需要干预内核就能直接在用户空间完全管理的队列。同样地，
每个存储控制器暴露多个虚拟存储接口控制器（VSICs） 。

每个VSISC提供独立的由硬件多路复用的存储控制队列。联系每个这样的虚拟接口卡(VIC)是队列和速率限制器。VNICs也提供适配器和VSICs提供虚拟存储区域。

队列：每个虚拟接口卡(VIC)包含多对的DMA队列为用户空间发送和接收。准确形式的这些VIC队列会依赖于这些IO接口发送接收。

发送和接收过滤器：一个发送过滤器是可以在网络包头域预测的，硬件将使用觉得是否发送或者丢弃。

虚拟存储区域：存储控制器需要通过物理函数映射虚拟存储区域到物理的驱动和相关的VSICs提供一个接口。典型的VSA将会是足够的大允许应用忽略底层的复用。

带宽分配：这包括自持资源分配机制例如速率限制和

####3.3. VSIC模拟

为了验证模型支持的存储设备，作者通过专门处理器核心开发VSIC支持原型来模拟期望的硬件功能。同样的方法也可以用来运行阿莱克斯系统没有VNIC支持

为了处理来自操作系统的IO请求，磁盘阵列控制器提供固定大小的一个请求和一个响应描述符队列，实现了圆形缓冲区，以及一个软件控制寄存器（PR）指向的请求描述符队列的头部。
请求描述符（rqds）有一个大小为256字节，包含一个SCSI命令，分散收集阵列系统内存范围和目标逻辑磁盘数量。SCSI命令指定操作类型（读或写），总的传输大小和磁盘上的基本逻辑块地址（LBA）。
散点集数组指定系统内存中的请求的相应区域。响应是指由描述符队列入口完成rqds包含完成代码。一个RQD只能收到响应后重复使用。

我们为每个VSIC复制设置分配队列和系统内存中同样的格式寄存器文件映射到应用程序和专用VSIC核心。像82599，我们限制VSiCs的最大数目64。
此外，该VSIC核心保持高达四个VSA映射每个VSIC，只可以从控制平面阵列进行编程。映射包含VSA的大小和LBA逻辑磁盘内的偏移，有效地指定范围。

####3.4. 控制面板接口

应用程序和Arrakis控制面板接口被用来从系统和直接的输入输出流从用户程序请求资源。接口展示的关键抽象是VICs ，doorbells,适配过滤器，VSAs和速率说明符。

应用程序会创建或者删除VICs和相应的doorbells和特定的VICs上的特定事件。doorbell是一个进程间通信端点用来通知应用程序事件已经发生，并且之后被讨论。
VICs是硬件资源因此Arrakis一定分配它们到应用中。

过滤器会用来指定IP地址和联系有效包在每个VNIC发送接收端口数量的范围。过滤器是一个为了我们的目的而不是传统连接定位符更好的抽象，因为它们可以编码大量的传输模式，
并且包含传统的端口分配和接口指定。

应用程序创建一个带有控制平面操作的筛选器。在一般情况下，一个简单的高级包装：filter= create_filter（flags，peerlist，serviceList）。
flags指定滤波器的方向（发送或接收）和过滤器是否指的是以太网，IP，TCP或UDP报头。peerlist列表是公认的通信节点根据指定的过滤器类型，
并列出接受serviceList包含服务地址（例如，端口号）的过滤器。允许使用通配符

调用create_filter返回一个内核过滤器filter，内核保护能力授权发送或接收匹配谓词分组，然后可以分配到一个特定的队列VNIC。VSA的获取和分配在一个类似的方式VSiCs。

同样地，一个虚拟存储区域可以获得通过一个acquire_vsa(name)=VSA调用。返回的VSA能允许应用程序访问VSA。控制面板存储能力，它们联系在分开的存储区域。如果不存在VSA被获取，
它的大小初始化为0，但是它通过resize_vsa调整大小。VSAs存储虚拟存储块映射到物理区域并且给予一个全局名，因此它们可以再被找到当应用程序重启。

查找文件名

在Arrakis中的设计原则是分隔文件名和实现。在传统的系统中，完全合格的文件名指定用于存储文件的文件系统和它的元数据格式。在Arrakis中提供应用程序直接控制VSA存储分配：
一个应用自由的使用它的VSA来存储元数据，目录和文件数据。允许其他的应用访问数据，应用会导出文件和目录名内核虚拟文件系统。对于VFS的其他部分，
一个应用管理文件或者目录像是一个远程安装点-一个简介方式实现文件系统。文件或者系统操作没有内核敢于就进行了本地处理。

其他应用程序可以访问这些文件的三种方式。默认情况下At=rrakis应用库管理VSA导出文件服务器接口；
其他应用程序可以使用普通的POSIX API调用通过用户级RPC嵌入式库文件服务器。该库还可以作为一个独立的进程来运行，以提供当初始应用程序不活跃时的访问。
就像一个普通的文件系统上的，库需要只实现对其VSA的文件访问功能和需要可以选择跳过任何POSIX它不直接支持特性功能。

####3.6. 网络数据面接口

在Arrakis里，应用通过直接用硬件交流发送和接收网络包。数据面接口因此在一个应用库中被实现，允许它在应用中一起被设计。Arrakis库给应用程序提供两个接口，
我们描述原生Arrakis接口，从POSIX标准分开来支持真正的零拷贝输入输出；Arrakis也提供一个POSIX兼容层支持没有修改的应用。

首先，分组之间在网络和使用传统的DMA技术使用的数据包缓冲区描述符环内存中异步传送数据包。

其次，应用程序将发送网络数据包的硬件由入队的缓冲器链到硬件描述环的所有权，并获得了相反的过程，接收到的数据包。这是由两个虚拟网卡驱动程序的功能进行。
send_packet（队列，packet_array）发送队列的包；包被分散规定收集阵列packet_array和必须遵守的队列已经相关滤波器。receive_packet（队列）=包从队列接收数据包并返回一个指向
它的指针。这两种操作都是异步的。packet_done（包）返回所有接收的分组的VNIC。

为了优化性能，Arrakis协议栈不通过这些调用而直接通过生成的编译器与硬件队列交互，优化代码定制为网卡描述符格式。

第三，使用和队列相关的doorbells处理异步事件通知。Doorbells通过硬件虚拟化中段直接从硬件传输到用户程序。

###存储数据平面接口

低级存储API提供一系列的命令异步读写和清洗硬件在任意偏移和在任何大小的虚拟存储区域通过一个命令序列在相联的VSIC。要这样做，调用者提供一个数组的虚拟内存范围（地址和大小），
这个VSA定义符，队列数，和匹配的VSA内的数组范围。实现相应的命令入队到VSIC，凝聚记录了命令如果这使得底层的多媒体有意义。输入输出完成事件使用doorbells来报告。在这之上，
提供了一个POSIX兼容的文件系统。

作者设计了一个持久数据结构库，Caladan，利用了低延迟的存储设备。持久数据结构比简单的文件系统提供的读写接口可以更加有效。缺点是对于POSIX API缺少向后兼容性。
持久数据结构的设计目标是操作立即持久，结构对于死机故障鲁棒，操作有最小延迟。

根据这些规则设计了持久日志和队列数据并且修改一些应用来使用它们，日志API包括操作打开和关闭日志文件，创建日志入口，将它们附加到日志上，通过日志迭代和修建。
API队列添加一个出栈操作去包括修改和读队列。持久是异步的，附加操作直接返回持久回调，允许我们掩码保持写延迟，优化准备网络回应客户端，当入口坚持。

###实现

Arrakis操作系统基于Barrelfish多核操作系统代码上实现。基于SR-IOV的支持，修改已有的PCI设备管理器去认出处理SR-IOV扩展PCI能力。实现Intel82599 10G以太网适配器驱动来初始化和管理一些虚拟函数。作者为82599实现一个虚拟函数驱动，包括支持扩展信息信号中断，可以被用来传输per-VNIC doorbell事件到应用程序。最后实现了intel IOMMU驱动。开发可自己的用户层网络协议栈，
Extaris是一个共享的库会直接与虚拟函数接口交互并且提供POSIX的socket API和Arrakis的原生API到应用

###局限性

由于限制滤波器支持82599NIC网卡，实现为每个VNIC使用了一个不同的MAC地址，我们直接使用流到应用然后在应用软件做了更多的细粒度的过滤。
更多通用滤波器可用性将消除软件的开销。实现的虚函数驱动现在不支持82599特性的传输描述符，减少了PCI必要总线传输的必要性，
期望看到一个5%的网络性能提升从添加的支持。

实验中使用的RS3 RAID控制器不支持SR-IOV或者VSAs。Hense,用来作为物理函数，提供一个硬件队列，映射一个VSA到每个控制器提供的逻辑磁盘，
仍然使用了IOMMU映射一个VSA到每个控制器提供的逻辑磁盘。

细粒度的输入输出调度。无中介设备设备访问需要一些输入输出调度在硬件中被实施。现在硬件技术提供相对粗粒度的调度，
例如一个比例限制和队列虚拟控制器输入输出层次。粗粒度调度，例如网络拥塞控制，留给单独的应用层输入输出协议栈。然而在多用户租用的数据中心，
不合适相信永远程序，或者甚至虚拟机它们自己的拥塞控制。这引起的问题是如何控制输入输出调度未被信任的输入输出协议栈。

IX操作系统为了再次保护应用层输入输出协议栈因此不能被应用开发者修改，第二种方式是增加更多粗粒度的输入输出控制到硬件输入输出设备，
提出了一个更复杂的网络设备可以用于执行粗粒度的输入输出调度，基于包匹配和行动范式。网卡可以用传输级别网络协议执行有限的迭代，
使用这种能力执行拥塞控制和其他的在硬件层粗粒度输入输出调度任务。另一方面，网络协议需要适当搭配和行动范式允许粗粒度的硬件输入输出调度。

第三种方式是整合Arrakis和一个分布式数据中心网络资源分配。早期工作显示分布式速率限制会为有效和公平的拥塞控制机制共享瓶颈提供基础，
分布式控制器也许可行，这种方法将整合灵活的硬件输入输出控制器提供高性能拥塞控制为相关多用户数据中心情形不可信的应用。

###评测

评测Arrakis在四个应用负载下，分别是：一个典型的在许多大型memcached分布式缓存系统观察到读取重载模式;
一个写重载模式到Redis持续性NoSQL存储；一个通过HTTP 负载均衡的web服务器；一个IP层次的中间件。
作者也测试系统在最大负载下面再一系列的基准程序并且在一些网络应用中分析性能串扰

通过安装最新的ixgbe设备驱动器和关闭接收方扩展调整Linux网络性能。RSS传播包到几个网卡接收队列中但是招致在单一核中不必要的相干。

###服务器端的包处理性能

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/16.jpg" style="width: 50%; height: 50%"/>​

重复的服务器能够在发送每个数据包之前添加一个可配置的延迟。使用这个延迟来模拟额外的应用程序级处理时间在服务器。
图7显示了平均吞吐量达到每一个系统在各种这样的延误；理论线损率1.26M PPS零处理。在最佳的情况下（没有额外的处理时间），
Arrakis/P到2.3倍的吞吐量。从POSIX，Arrakis / n达到3.9×Linux的吞吐量。Arrakis的相对优势消失在64μs评估如何接近Arrakis最大可能的吞吐量，
我们嵌入式最小服务器直接进入NIC设备驱动程序，消除任何剩余的API开销。Arrakis / n到94%的驱动程序限制。

###memched键值对存储

memcachedArrakis/ P达到1.7×Linux核心的吞吐量达到在四CPU核心线性关系。在所有六核更低的吞吐量是由于与Barrelfish系统管理过程的争夺，
如内存和进程管理，持续投票内核之间通讯渠道和在核心，造成高CPU利用率。相比之下，Linux的吞吐量平稳时间超过两核。
一个单一的多线程的缓存实例表明吞吐量没有明显改善处理方案。这并不奇怪，因为memcached是规模规模。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/17.jpg" style="width: 50%; height: 50%"/>​

Redis的缓存模型从缓存到一个持久的NoSQL对象存储延伸。我们的研究结果表II表明操作虽然比memcached更费力，Redis仍由I/O栈的开销占主导地位。
Redis可以用在Memcached同一个场景，我们都遵循着相同的实验装置，使用Redis版本2.8.5。我们使用基准测试工具使用和配置分布式执行get和set在两个单独的基准要求65536范围内的随机密钥1024个字节的值的大小，
坚持每一set操作，共有1600对并发连接的客户机执行redis是单线程的，所以我们只有单核心的性能研究。

Redis的Arrakis版本使用Caladan。在管理和与Caladan记录交换记录而不是一个文件的Caladan日志应用我们改变109线。
我们不排除使用的Redis封包处理开销。如果我们做了，我们会保存另2.43us的写延迟。由于快速I/O栈，Redis的读取性能与memcached，
写延迟提高了63%的延迟，而写吞吐量极大地提高了9×倍。

###HTTP负载均衡

探讨性能的影响时，很多关系需要维护，我们配置五个Web服务器和一个负载均衡器。为了最大限度地减少网络服务器的开销，
我们部署了一个简单的静态网页1024个字节，使用主存。这些相同的Web服务器主机也可作为负载产生，使用ApacheBench工作量，
2.3版本进行尽可能多的请求并发的网页。每个请求封装在自己的TCP连接。在负载平衡器的主机，我们部署HAProxy版1.4.24，配置以循环的方式来分配负载。
我们运行在负载均衡节点HAProxy过程的多个副本，每个执行自己的端口。我们配置ApacheBench实例来发布他们的负载均分可用HAProxy实例。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/18.jpg" style="width: 50%; height: 50%"/>​

###总结

在这篇文章中，作者描述和评估Arrakis,，一个新的操作系统，旨在从内核中删除输入/输出数据路径，而不影响进程隔离。
不同于传统的操作系统，介导所有的输入/输出操作，执行过程隔离和资源限制，Arrakis使用的设备硬件提供输入输出直接到一个自定义的用户级库。
Arrakis内核运行在控制平面，配置硬件限制应用程序的不当行为。

为了作者我们的方法的实用性，作者已经实现了阿莱克斯在商业上可用的网络和存储硬件，并用它来衡量几个典型的服务器的工作负载。
作者实验能够表明，保护和高性能并不矛盾：Arrakis比一个调谐的Linux实现在端到端的客户端读取和写入延迟到redis持久的NoSQL存储2–5倍快，
写吞吐量9倍。
