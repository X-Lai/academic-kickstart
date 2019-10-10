---

title: Fast R-CNN

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-07-10"
tags: ["object detection"]

---

# Fast R-CNN

相比于classification，object detection任务准确度更低，难度更大，主要由于两个挑战：

1.需要处理大量的proposal；

2.在region proposal的rough localization的基础上需要refine以获得准确的localization。

---

## R-CNN/SPP-net的缺点：

1.R-CNN和SPP-net都是multi-stage的；

2.R-CNN运行慢，而且训练时耗费时间和空间；

3.SPP-net在fine-tune时不会更新spatial pyramid pooling之前的网络参数。

---

## Architecture

和SPP-net类似，Fast R-CNN将整个图片作为输入，得到feature map，并得到每个region proposal在feature map中对应的window，经过RoI pooling layer得到长度固定的向量。之后，每个特征向量都会经过一系列fc层，最终分为两支：1）softmax层classifier，2）refined bounding-box position regressor。

### RoI pooling layer

将h$\times$w的RoI window分割成H$\times$W grid of sub-windows（H和W都是hyperparameters），每个sub-window的大小约为h/H$\times$w/W，每个sub-window都通过max pooling得到一个response。每一个channel独立进行pooling。最终不同size的window都能输出成size为H$\times$W的向量。

### Transformations of pre-trained networks

从pre-trained network到新的模型需要经历3个transformations：

1.最后一个max-pooling layer换成RoI Pooling layer;

2.1000-way classifier要换成(K+1)-way classifier和bounding-box regressor;

3.输入有两个：1）图片，2）RoI

### Fine-tuning

SPP-net在spp layer之前的层中的weights在fine-tuning的时候是不会更新的，这就有可能会限制识别的准确度。但是为什么不会更新呢？关键在于同一个batch中的sample（即SPP-net的RoI window）会来自于不同的图片，如果要更新在spp layer之前的weights，那么就需要首先得到每个sample对应的RoI window的receptive field。而一个window的receptive field会非常大，甚至常常是整个图片，所以当同一个batch的sample来自不同的图片时，在一个batch训练过程中就需要输入很多张图片，这会导致极度的inefficiency。

本文提出了一种更加efficient的训练方法来改善这个问题：hierarchical sampling。即在训练的时候，为了得到一个batch，先sample N个images，然后再从其中每个image中sample R/N个RoI（R是batch size）。当N越小，一个batch的计算量就会越小（因为处理的图片少了，共享的计算多了）。在本文中取N=2，R=128。

此外，和R-CNN、SPP-net不同的是，Fast R-CNN不需要分multi stages，只需要一个stage就能同时优化softmax classifier和bounding-box regressor。具体有：

*Multi-task loss*

$L(p,u,t^u,v)=L\_{cls}(p,u)+\lambda[u \ge 1]L_{loc}(t^u,v)$

Multi-task loss如上式所示。其中$L\_{cls}(p,u)=-log{p\_u}$，$L\_{loc}(t^u,v)=\sum\_{i \in \{x,y,w,h\}}{smooth\_{L\_1}(t\_i^u-v\_i)}$，$smooth_{L_1}(x)=\left\\{\begin{aligned}
& 0.5x^2  & if \  |x| < 1 \\\ & |x|-0.5  & otherwise
\end{aligned}
\right.$。

$p$是一个K+1维向量，表示将该window分到K类中每一类的概率（此外还需再加上background类）。$u$和$v$分别表示该window被标注的ground-truth的类别和被标注的ground-truth的四元组$(x,y,w,h)$，$t$表示Fast R-CNN使用该window预测得到的各个类别对应的四元组，$t^u$表示类别$u$对应的四元组$(t_x^u, t_y^u,t_w^u,t_h^u)$。

### Scale invariance

有两种方式实现scale invariant object detection：1）brute force：每张图片都以一个预先设定的size作为输入进行处理，2）multi-scale approach：在test的时候，使用image pyramid来scale-normalize每个object proposal，在multi-scale training的时候，在每次采样图片的时候，随机选择一个pyramid scale进行训练（也作为一种data augmentation的方式）。

---

