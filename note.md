---
layout: post
title: notebook
categories:
- paper
tags:
- technology
- 
---

##c++学习使用记录
 
 最近编写c++程序  使用opencv库经常出现c++内存泄露的问题 ，
 比如使用cvCopyImage是一个硬考贝过程  拷贝之后记得释放着两块内存
 
 另外就是命名冲突的问题，注意程序的模块化，使用命名空间进行编程
 
 另外就是把一些编程的坏习惯改掉，比如，总为了图方便使用全局变量
 
 在函数调用时候使用返回值
 
 另外，注意使用类，命名空间进行解耦
 
 还有就是重构！！！不要一个函数里面写一堆代码，写错了就删除
 
##hog
 
Dalal提出的Hog特征提取的过程：把样本图像分割为若干个像素的单元（cell），
把梯度方向平均划分为9个区间（bin），在每个单元里面对所有像素的梯度方向在各个方向区间进行直方图统计，
得到一个9维的特征向量，每相邻的4个单元构成一个块（block），把一个块内的特征向量联起来得到36维的特征向量，
用块对样本图像进行扫描，扫描步长为一个单元。最后将所有块的特征串联起来，就得到了人体的特征。
例如，对于64*128的图像而言，每16*16的像素组成一个cell，每2*2个cell组成一个块，因为每个cell有9个特征，
所以每个块内有4*9=36个特征，以8个像素为步长，那么，水平方向将有7个扫描窗口，垂直方向将有15个扫描窗口。
也就是说，64*128的图片，总共有36*7*15=3780个特征。


##sift

计算keypoint周围的16*16的window中每一个像素的梯度，而且使用高斯下降函数降低远离中心的权重。

<img src="https://github.com/dengguang2012/paper-Reading-Report/blob/master/illustraction/29.jpg" style="width: 50%; height: 50%"/>​

在每个4*4的1/16象限中，通过加权梯度值加到直方图8个方向区间中的一个，计算出一个梯度方向直方图。
这样就可以对每个feature形成一个4*4*8=128维的描述子，每一维都可以表示4*4个格子中一个的scale/orientation. 
将这个向量归一化之后，就进一步去除了光照的影响。

###ubuntu下面配置shadowsocks

通过PPA源安装，仅支持Ubuntu 14.04或更高版本。

	sudo add-apt-repository ppa:hzwhuang/ss-qt5
	sudo apt-get update
	sudo apt-get install shadowsocks-qt5

### puty远程访问ubuntu

	sudo apt-get install openssh-server  
	启动SSH服务：
	sudo /etc/init.d/ssh start  

SSH的服务端口，默认端口是22，此时就可以使用putty远程登录ubuntu了。

### 社会心理学阅读感悟

你的决定可能不是你真的想要的东西
 
 自我控制 自信乐观 抑郁或者苦恼的人变得被动是因为它们自己的努力没有起到任何作用。无助的狗和抑郁的人遭遇了意志瘫痪，被动顺从，甚至死气层层的冷漠
 
 efficiency and control 相信自己有能力和效率的人以及那些内控的人，比那些习得性无助和悲观绝望的人会应对的更好，取得更大的成就。
 
 与自尊脆弱的人相比，把自尊建立在良好的自我感觉而不是分数外貌金钱或者别人的赞美的基础上的自尊感明确的人，会一直感到状态良好
 
 与自尊建立在如个人品质这样的内部因素的人相比，自尊主要建立在外部因素基础上的自我价值感脆弱者会经历更多的压力、愤怒、人际关系问题、吸毒酗酒以及饮食障碍。哪些试图通过变得漂亮、富有或者收人欢饮来寻求自尊的人，对真正利于提高生活质量的东西视而不见
 进一步讲，如果良好的自我感觉是我们的目标，如果良好的自我感觉是我们的目标，我们就不会把批评放在心上，我们更加倾向于在压力中追求成功而不仅仅在行动中获得快乐
 
 老子：是以圣人去甚 去奢 去泰  所以明智的人去除身份 奢华 骄纵
 
 要想在学校里获得成功和出类拔萃，既需要足够的乐观精神以支撑希望，同时也需要足够的悲观心态以激起对利害的关注  即乐观而又不盲目乐观
 
 我们并不是客观的看待事物  而是总是从自己的角度出发来看待事物
 
 如果我们没有自我服务偏见，那谦逊呢？
 
 虚伪的谦逊其实是为了掩饰个体认为自己真的优于众人的想法
 
 自我服务偏见 虚伪的谦逊和自我妨碍 揭示出个体十分在意自我形象。在不同程度上，我们始终在管理自己给他人因遭的形象。我们总是向着周围的观众表演
 
 windows 下面 python 下面pip使用 安装
 
 下载 https://bootstrap.pypa.io/get-pip.py
 
 运行 get-pip.py
 
 关键是安装以后记得加环境变量 C:\Python27;C:\Python27\Scripts
 
 C:\Python27\Scripts 中有pip.exe
 
  
 ### windows 远程访问windows
 
 远程桌面连接
IP 用户名 密码

### python下面连接使用 mysql 

下载MySQL_python-1.2.5-cp27-none-win_amd64.whl

使用pip install pymysql 安装

连接pymysql

	import pymysql
	conn = pymysql.connect(host='localhost', port=3306,user='××××',passwd='×××',db='××××',charset='UTF8')
	cur = conn.cursor()
	cur.execute("INSERT INTO `test`(`testPri`, `name`, `birthData`, `sex`)  VALUES (1,'dengzhihui', '1993-12-05', 'm')")
	conn.commit()
	cur.close()
	conn.close()
