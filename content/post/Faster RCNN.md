---
title: Faster R-CNN

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-07-12"
tags: ["object detection"]
---

# Faster R-CNN

Fast R-CNN和SPP-net的速度相比于R-CNN而言，已经快了很多，但是仍然还有提升的空间，而且它们的时间瓶颈在于Region proposal上的耗时。所以Faster R-CNN提出使用RPN(Region Proposal Network)来得到region proposal，并且该RPN和detection network有一部分共享的convolutional layers，避免了feature map的重复计算，自然也就提高了速度。

------

Faster R-CNN分为两个modules：1）RPN，2）Fast R-CNN detector。

## Region Proposal Networks

RPN将一张任意size的图片作为输入，输出一个rectangular object proposal的集合，而且每个object proposal都附带一个objectness score（用以表示该proposal是object的程度）。

### anchor

之前说到RPN和detection network有一部分共享的convolutional layers，所以当图片输入这部分layers之后，会输出一个feature map。为了得到region proposals，本文提出了anchor的设计。使用一个small network在这个feature map上slide。这个small network将n$\times$n（本文取n=3）的spatial window作为输入，每个spatial window将会被映射到一个低维的特征向量。这个向量将用于两个sibling fc layer：1）box-regression layer，2）box-classification layer。每个spatial window对应一个anchor，每个anchor有k种以该anchor location为中心的anchor-box（在本文中设定$k=3\ scale\times3\ aspect\ ratio=9$），box-classification layer输出一个2k维向量（即对应k个anchor-box是或不是object的score），用来判断对应anchor-box是否为object proposal。另外box-regression layer输出一个4k维向量（即对应k个anchor-box的四元组），用来对anchor-box的位置进行refine。

这种anchor的设计，使得即便是single-scale的效果也是很好的，这样就大大减少了运行耗时。

### Loss function

在训练RPN时，需要确定每个anchor-box的label（即每个anchor-box是否包含object）。本文采用如下规则。将以下两种anchor标记为positive：1）与某一ground-truth box的IoU overlap最高的anchor，2）与某一ground-truth box的IoU overlap超过0.7。而与所有ground-truth box的IoU overlap小于0.3的anchor将被标记为negative。

training的目标是最小化：$L(p, t)=\frac{1}{N_{cls}}\sum\_i{L\_{cls}(p\_i,p\_i^* )}+\lambda\frac{1}{N\_{reg}}\sum\_i{p\_i^* L\_{reg}(t\_i,t\_i^* )}$

其中$p$表示classification layer输出的向量，每一维表示对应anchor是或不是object的概率。$p\_i^* $表示label值，若对应anchor含object的话，则值为1，否则为0。$N\_{cls}$表示mini-batch的大小（在本文中取256），$N\_{reg}$表示anchor location的数量（在本文中约为2400），$\lambda$在本文中取10，$L\_{cls}$表示是或不是object的log loss function，$L\_{reg}=smooth\_{L\_1}(t\_i-t\_i^* )$。$t$表示regression layer输出的向量，$t=(t\_x,t\_y,t\_w,t\_h)，t^* =(t\_x^* ,t\_y^* ,t\_w^* ,t\_h^* )$，具体定义如下：

$t\_x=(x-x\_a)/w\_a,t\_y=(y-y\_a)/h\_a,t\_w=log(w/w\_a),t\_h=log(h/h\_a)$

$t_x^* =(x^* -x_a)/w_a,t_y^* =(y^* -y_a)/h_a,t_w^* =log(w^* /w_a),t_h^* =log(h^* /h_a)$

其中$x,y,w,h$分别表示box的中心点坐标、宽度和高度，$x,x_a,x^*$分别表示预测的box、anchor的box和ground-truth的box（对$y,w,h$类似如此）。

### Training RPN and Fast R-CNN detection network

由于RPN和detection network共享了一部分convolutional layers，所以在训练的时候，如果两部分network分开独立训练的话，不同的方式会有不同的效果。而且如果想要将这两部分network同时训练back propagate，有一个问题，即输入进detection network的bounding box coordinates实际上是输入图片的一个函数，而并非像之前使用selective search得到region proposal一样是固定的，所以如果要back propagate的话，要计算loss对bounding box coordinates的偏导，这是nontrivial的，将会浪费很多训练时间在back propagate上。所以为了加快训练的速度，将两部分network分开独立训练，从而得到一个可以接受的近似解才是现实的。

本文提出一种alternating training。首先训练RPN，然后再使用当前已训练的RPN来获取proposal，并以此来训练Fast R-CNN的detection network，fine tune Fast R-CNN，然后使用Fast R-CNN的参数来初始化RPN，然后继续迭代这个过程。直到收敛。

------

