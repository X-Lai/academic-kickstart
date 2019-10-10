---
title: SSD
# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-09-12"
tags: ["object detection"]
---

# SSD: Single Shot MultiBox Detector

和YOLO一样，SSD也是一种one-stage的object detector。SSD将prediction bounding boxes的输出空间离散化到多个feature maps上每个位置的多个不同scale和aspect ratios的default boxes上。

## Framework

SSD在训练时只需要输入图片和对每个object的gt boxes。SSD将会在不同resolution的feature maps上来预测每个location使用哪个scale和aspect ratio的default boxes。对于每个default box，SSD将会预测shape offsets以及object category confidences（包括background类）。在训练的时候，先将这些default boxes match到gt boxes上，确定哪些default boxes是positive（即被认为是包含object的），剩余的default boxes就被作为negatives。然后再结合prediction来计算loss，进行训练。

## 如何match default boxes

与YOLO v2不同的是，SSD提出不一定要对每个gt box只match一个default box(anchor box)。SSD先将每个gt box match到与它自己IOU最大的那个default box上，然后再将那些与任一gt box的IOU超过某个threshold（本文设为0.5）的default boxes也标记对应到该gt box/objectness/class。

## Multi-scale feature maps for detection

在预测的时候，SSD使用多种不同scale的feature maps来进行detection。

## Convolutional predictor for detection

对于所有不同scale的feature map，都将采用若干个3*3的filter来进行卷积，从而进行预测。

## Default boxes and aspect ratios

对于feature map的每个cell的每个default box，SSD都会预测offsets以及classification。这样每种feature map就会有$(c+4)k$个filters（假设一共有c-1个class，另外加上background共有c个class）。其中k为default boxes的个数（本文取6）。

## Training

### Training Objective

根据之前‘如何match default boxes’所述，我们可以获得哪些default boxes是positives或negatives。随后即可计算loss function。$L(x,c,l,g)=\frac{1}{N} (L\_{conf}(x,c)+\alpha{L\_{loc}(x,l,g)})$。其中$x\_{ij}^k$要么取1，要么取0，表示第i个default box是否和第j个gt box(其category为k) match，$c,l,g$分别表示预测的class，预测的box以及预测的gt box。$L\_{loc}$是使用和faster R-CNN类似的localization error。而$L\_{conf}(x,c)=-\sum\_{i\in Positives}^{N}x\_{ij}^plog(\hat{c\_i^p})-\sum\_{i\in Negatives}{log(\hat{c_i^0})}, \hat{c_i^p}=\frac{exp(c_i^p)}{\sum_p{exp(c_i^p)}}$。

### Choose scales and aspect ratios for default boxes

设共有m个不同scale的feature maps，那么每个feature map的default boxes的scale就用如下公式计算：$s\_k=s\_{min}+\frac{s\_{max}-s\_{min}}{m-1}(k-1),\quad k\in[1,m]$。本文取$s\_{min}=0.2, s\_{max}=0.9$。另外再取aspect ratios。$a_r\in \\{1,2,3,\frac{1}{2},\frac{1}{3}\\}$，第k个feature map的default box的宽$w_k^a$和高$h_k^a$分别用$w\_k^a=s\_k\sqrt{a\_r},h\_k^a=s\_k\sqrt{a\_r}$来计算。另外对于aspect ratio=1的情况，还要再加一种size：$s\_{k}'=\sqrt{s\_ks\_{k+1}}$。

### Hard negative mining

对所有negative default boxes对应的confidence按高到低排列，然后取top的那些negatives，使得negatives和positives的比例最多3:1。

### Data augmentation

本文针对object detection提出一种新的data augmentation的方法，sample a patch。sample完毕之后，对于每个gt box，如果它的中心仍然在这个patch中，则保留这个gt box。

## 和YOLO v2的不同点

1. Gt boxes matching strategy不同；
2. 使用multi-scale feature maps的方式不同；
3. 使用了新型的data augmentation方法；
4. default boxes或anchor数量不同；
5. backbone network也不同；
6. 使用了hard negative mining;
7. 预测offsets的方式不同。



