---
layout: post
title: Non-scalable locks are dangerous
categories:
- technology
tags:
- paper
- 
---

##Non-scalable locks are dangerous

Non-scalable locks are dangerous 阅读报告

###背景介绍

多个操作系统依赖不可扩展旋转锁用于序列化。
例如，Linux内核使用的是ticket spin locks，即使是可扩展锁有更好的理论特性。
在48核机器上使用linux得出非可扩展的锁可能会导致实际负载性能崩溃，即使是非常短的关键部分。崩溃突发和本质可以使用基于markov性能模型进行解释。
可扩展的自旋锁取代得不可伸缩的自旋锁可以避免崩溃，而只需要适度的修改源代码。

不可扩展的锁，比如简单的spinlock，在大量冲突的情况下性能差，我们已经遇到了多个情况下，系统吞吐量突然由于非可扩展的锁崩溃：例如，一个系统，
在25个核情况下执行不错在30核下就完全崩溃。同样令人惊讶的是，这些违规的关键部分往往是很小的。本文论证非可扩展的锁危险性。具体而言，它主要针对Linux内核锁。

内核突然增加导致性能崩溃，当然，一个特别是在操作系统内核的工作负载和竞争水平下应该使用可扩展的锁是很难控制。作为示范，
作者将Linux的spinlock替换成可扩展的MCSlock，再运行造成崩溃的软件性能，4个基准中3个基准的内核改变简单的。对于第四种情况，
由于目录缓存使用了一个复杂的锁定方案，而一般的目录缓存是复杂的，因此该更改更为复杂。MCS锁大大提高了可扩展性，因为它避免了性能崩溃。
作者尝试其他可扩展的锁实验，包括分层的，观察到的改进是可以忽略不计，从非可扩展的锁到可扩展的锁性能才有很大的改进，
这种方法目标是不可扩展的行为应该是由修改软件来消除序列化的瓶颈，而可扩展的锁只是推迟了这种修改的需要。然而在实践中，消除在内核中所有潜在的竞争是不可能的，
即使内核在某个时间点有可伸缩性，同样的内核很可能会在随后的一代硬件上有缩放的瓶颈。查看可伸缩锁的一种方法是应用更基本的缩放改进到内核的来缓解关键时刻。

从以前的非可扩展的锁有风险工作中总结得出结论，：他们不仅性能差，而且可能会导致整体系统性能崩溃。更具体地说，
首先，作者证明了不可伸缩的锁性能差在实际工作中导致性能崩溃，即使自spin lock是在保护内核很短的关键部分。
其次，作者为提出了一个单一的能充分捕捉所有的操作系统的综合模型。
第三，我们在现代基于x86的多核处理器上面确认MCS locks可以提高最大可扩展性而不降低性能，并得出这样的结论：不同尺度可伸缩的锁伸缩和性能优势很小。

###论证不可扩展锁的问题

非可扩展的spin locks导致一些内核密集型工作负载的性能崩溃。我们从四个基准证明在一个核上面关键部分消耗低于1%的CPU周期，
在48核的x86机器上面可能导致性能崩溃。

具体讨论了Linux内核的ticket lock，但任何类型的不可扩展锁将展示这一部分中所示的问题。图介绍了简化的C++代码。ticket socket是内核版本2.6.25后默认锁

	struct spinlock_t {
	 int current_ticket ; 
	int next_ticket ; 
	} 
	void spin_lock ( spinlock_t *lock) { 
	int t = 
	atomic_fetch_and_inc (&lock -> next_ticket ); 
	while (t != lock -> current_ticket ) ; /* spin */
	 } 
	void spin_unlock ( spinlock_t *lock) { 
	lock -> current_ticket ++;
	 }
	 
得到ticket number，内核采用next_ticket原子增量指令。核心然后旋转，直到它的机票号码是当前。释放锁，current_ticket核心增量，使锁交到核心，等待下一次的号码。
如果很多内核都在等待一个锁，它们将全部锁定缓存变量。解锁将使得缓存入口变得无效的。所有的内核此时读取高速缓存行。在大多数结构，
读取是序列化的（要么通过共享总线或高速缓存行的根目录或目录节点），因此完成他们需要花费核的数量比例的时间。该锁的核是在下一行，
该锁可以期望通过这个过程来接收它的缓存行的副本。因此，每一个锁的切换开支随着等待内核的数量增加也成本的增加。每一个核的运作需要一百个周期的顺序，
所以一个单一的发行可以采取许多数以千计的周期，如果几十个内核等待。简单的测试和设置每次释放spin lock产生了类似的o(n)的成本。

测试的基准使用了FOPS，mempop, pfind和exim。

图显示了所有基准的结果。人们可能会期望总吞吐量在一段时间内与内核的数量成比例上升，然后再由于一些序列部分瓶颈持平。吞吐量随多个内核的增加而增加，
然而在一定数量内核之后突然降低，N个内核的性能在一到两个更多地内核的时候却更好。

<img src="https://github.com/dengguang2012/paper-Reading-Report/illustraction/2.jpg" style="width: 50%; height: 50%"/>​

###模型

本节提出了一种不可伸缩的锁的性能模型，解答在上一节中提出的问题。首先介绍了在一个较高的水平硬件高速缓存一致性协议，这是代表一个典型的x86系统的，
然后基于此协议的基本性质构建了解性能模型,该模型可以预测观察到的崩溃行为。

硬件缓存相关性

模型假设基于目录缓存相关协议，所有的目录直接连接到目录内的网络。缓存相关性协议是AMD Opteron和Intel Xeon CPUs 的实现简化。

	static void anon_vma_chain_link(struct anon_vma_chain *avc , 
	struct anon_vma *anon_vma) 
	{ 
		spin_lock (& anon_vma ->lock );
		 list_add_tail (&avc ->same_anon_vma , &anon_vma ->head ); 
		spin_unlock (& anon_vma ->lock );
	 } 
	static void anon_vma_unlink ( 
	struct anon_vma_chain *avc , 
	struct anon_vma *anon_vma) 
	{ 
		spin_lock (& anon_vma ->lock ); 
		list_del (&avc -> same_anon_vma ); 
		spin_unlock (& anon_vma ->lock );
	 }

至于ticket locks的性能模型，为了理解在ticket-based spin locks 观察到的崩溃。构建一个准确的spin lock行为模型基于有两种运行机制：当没有竞争时，the spin lock可以很快的被获取，但是当许多内核尝试同时获取锁，传递锁的所有权花费的时间增加了很多。此外，锁改变行为确切的点依赖于锁使用模型和其他参数下面关键区域的长度。为了对于锁行为建立精确的模型，
作者使用队列理论把ticket lock建模为一个markov链。链中不同的状态表示不同熟练的核排队等待一个锁，模型中一共n+1个状态，表明我们的系统有固定数量的内核(n)


<img src="https://github.com/dengguang2012/paper-Reading-Report/illustraction/3.jpg" style="width: 50%; height: 50%"/>​

不同状态之间到达和服务概率表示锁的获取和释放。每一对状态这些概率是不同的，对ticket lock的非扩展性能和系统关闭的事实进行建模。特别地，到达概率从k到k+1等待，a_k应该是保留核（已经不在等待锁）数量的某个比例。
相反，服务概率从k+1到k，s_k应该与k成反比，反映的事实是传递对ticket lock的控制到下一个核花费等待数量线性时间。

为了计算到达比率，定义a是在单个内核连续获取锁的平均时间，单个核上面将试图获取锁的比率，在没有竞争的情况下，是1/a。因此，如果k个核已经在等待锁，新的竞争到达比例是a_k=(n-k)/a，因为不需要考虑已经在等待锁的内核。

为了计算服务比例，定义了两个参数：s，在串行区域花费的时间，c，在根目录下回应缓存线的时间。在缓存相关规则中，缓存线根目录轮流回应缓存线的请求。因此如果从不同内核有k个请求获取锁的缓存线，
直到赢家获取到缓存线时间将平均为c*k/2 。结果处理序列区域和传递锁给下一个持有者，当k个内核冲突花费了s+ck/2，并且服务概率为s_k=1/(s+ck/2)
当Markov模型准确的表示了ticket lock的行为，它并不能匹配任何能为队列建模提供简单公式的标准序列理论。特别是在系统关闭时，服务时间随着序列大小不同而改变。
为了计算这个公式，作者使用第一原则继承。让P_0,…,P_n是分别在状态0到n锁的稳定状态的概率。稳定的状态意味着转移概率的平衡：
	
	P_k∙a_k=P_(k+1)∙s_k，则P_k=P_0∙n!/(a^k (n-k)!)∙

<img src="https://github.com/dengguang2012/paper-Reading-Report/illustraction/0.JPG" style="width: 50%; height: 50%"/>​

为锁冲突核的每个数量给出稳定状态的概率，可以按照期待分布数值计算等待核的平均数量，加速达到锁的存在和序列化区域会计算为n-w，因为平均下来许多核在做有用的工作而w核在自旋。

###模型验证

为了验证模型，图8和图9显示用单一锁的microbenchmark预测和实际加速，花费固定数量的周期数而不是在一序列由锁保护的区域。图8显示了预测和实际的加速时的串行部分总是需要400个周期来执行，
而非串行部分的从12.5K 200K周期范围变化。正如我们所看到的，与所有配置的模型密切匹配实际硬件加速。

<img src="https://github.com/dengguang2012/paper-Reading-Report/illustraction/1.JPG" style="width: 50%; height: 50%"/>​

