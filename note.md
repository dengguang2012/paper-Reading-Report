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

ubuntu下面配置shadowsocks

通过PPA源安装，仅支持Ubuntu 14.04或更高版本。

	sudo add-apt-repository ppa:hzwhuang/ss-qt5
	sudo apt-get update
	sudo apt-get install shadowsocks-qt5

	