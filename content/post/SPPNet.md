---
title: SPPNet

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


# SPPNet：spatial pyramid pooling network

## SPP

spatial pyramid pooling：现存的cnn几乎都要求输入图片是固定size的。这种要求是人为的，而且可能会降低对图片或者任意size的子图片的识别准确度。SPP-net可以不管图片的size/scale都生成一个固定长度的向量。而且pyramid pooling还对物体变形非常健壮。

对于一个conv层的输出（即多个filter对应的feature maps），对其采用spatial pyramid pooling，对每个feature map都使用多个bin（bin的scale随feature map的size而变化，从而使得不论feature map的size如何，number of bins都不变），每个bin覆盖的区域都将输出一个response。所以经过spp能够输出固定长度的向量。

---

以往对于size不同的图片输入cnn，解决办法是使用warp或者crop，但是这可能会导致object的distortion或content loss，从而导致物体识别准确度下降。本文提出cnn对输入图片size固定的要求仅仅是由于fc层或classifier层导致的，conv层的卷积核filter是sliding window的，所以用于提取某种pattern，并不要求输入size是固定的。所以本文提出一种新的网络结构，在fc层之前加入spatial pyramid pooling，从而使网络不再要求输入图片的size固定。

---

## spatial pyramid pooling layer的好处

1.输入图片可以是任意scale的，所以在训练时，可以resize输入的图片，将输入图片转化成任意scale，再输入到相同的网络中。这样会使该网络能够提取在不同scale下的特征；

2.the coarsest pyramid level（即对应spatial pyramid pooling layer中bin能够占据整个image的情况）实际上是一种global pooling操作，在其他工作中，被发现具有减少overfitting和提高准确度的作用，并且global max pooling被认为有助于weakly supervised object recognition.

---

## SPP-net for object detection

在object detection任务上，相对于以前的方法，R-CNN准确度提升了很多。但是R-CNN是从原始图片中选取～2000个region proposals，所以对于每个region proposal都需要经过cnn提取feature，这非常耗时。sap-net提出先将原始图片经过一个cnn得到整体的feature map，再从feature map中提取~2000个region，并接着使用spatial pyramid pooling对不同size的region得到相同维度的向量。所以spp-net能够大大减少前向传播的时间。

### multi-scale feature extraction

使用multi-scale feature extraction可以进一步优化。本文提出resize图片到多个scale得到image pyramid，使$min(w,h)=s\in S=\\{480,576,688,864,1200\\}$（使用min函数可以保证feature window足够大）。再将此image pyramid作为cnn的输入得到feature maps。如何将这些不同scale的feature maps结合起来有两种方式。1）将这些不同的scale的feature maps堆叠在一起，然后channel-by-channel pooling到固定维度的向量，2）只选取其中一个scale使得scaled后的candidate window的number of pixels最接近224$\times$224，只需要使用这一个scale对应的feature map计算feature window并pooling到固定维度的向量用于训练即可。

---

