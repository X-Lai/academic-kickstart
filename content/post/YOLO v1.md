---
title: YOLO v1

# View.
#   1 = List
#   2 = Compact
#   3 = Card
view: 2

# Optional header image (relative to `static/img/` folder).
header:
  caption: ""
  image: ""
  
date: "2019-09-08"
tags: ["object detection"]
---

# YOLO: You Only Look Once

以往的目标检测器都是分成多个部分进行处理的，本文提出一种新的目标检测框架YOLO，通过单个神经网络直接根据full images来预测bounding boxes和class probabilities。

---

YOLO框架有三个优点：

1.这个架构非常快；

2.YOLO虽然产生更多的localization errors，但是更少的false positive on background（即很少将background预测成object）。因为不像sliding windows或者region proposal-based的方法，YOLO是在整张图片上全局地进行预测，所以它会implicitly encodes context information，从而提高分类的准确度。

3.YOLO可以学到非常general的representations of object。

## Unified Detection

该系统首先将输入图片划分成$S \times S$的grid，如果一个object的中心落到了一个grid cell里面，那么该grid cell就负责检测该object。每个grid cell会预测B个bounding boxes以及对应的confidence scores，这些confidene scores反映了其分别对应的bounding box包含物体的可能性以及这个bounding box预测的准确性。所以本文将confidence定义为$ Pr(Object) * IOU^{truth}\_{pred} $。其中$ IOU^{truth}_{pred} $表示当前预测出的bounding box与ground-truth bounding box的IOU。

### 训练的时候如何得到ground truth：

每个bounding box包括5个预测值：x, y, w, h, confidence。先判断哪些grid cells中落入了object的中心，若某个grid cell有object，它的ground-truth confidence就是$IOU^{truth}_{pred}$，若某个grid cell没有object，那么它对应的ground-truth confidence就是0。此外，(x, y)是ground-truth的bounding box的中心相对于grid cell的边界的比例。w和h分别是ground-truth的bounding box相对于整个图片宽和高的比例。

### Testing：

此外，每个grid cell还预测C个conditional class probabilities, $Pr(Class_i\|Object)$。不管B有多大，每个grid cell只会预测one set of class probabilities。

在test的时候，将conditional class probabilities和confidence predictions相乘即可得到class-specific confidence scores。因为$ Pr(Class_i\|Object)\*Pr(Object)\*IOU^{truth}\_{pred}=Pr(Class_i)\*IOU^{truth}\_{pred} $。

最终，对预测值进行nms，并过滤confidence score小于某阈值的prediction，得到最终目标检测结果。

### Training:

YOLO在预测的时候，每个grid cells会预测B个bounding boxes，但是在训练时，只挑选其中之一来负责预测某一个object（对存在object的grid cells而言）。而挑选的依据就是挑选当前与ground-truth bounding box的IOU最大的那个（每次训练时不一定固定，是由当前输出动态决定的）。本文提出这样做会造成bounding box predictions之间的specialization（即相同的grid cell对应的不同predictors分别负责不同的sizes/aspect ratios/classes），提高整体的recall。

#### Loss Function

使用pretrained ImageNet model进行fine tune。并且在设计loss function时，没有直接使用sum squared error。因为：

1.sum squared error简单地将localization error和classification error的权重设为相同了；（改变coordinates的weight）

2.在每个image中，有很多grid cells都不包含object，那么训练时会将这些grid cells的confidence scores都推向0，而这些grid cells的gradient常常会超过其他那些包含object的grid cells的gradient。所以这就导致了模型的不稳定性，使得训练在很早的时候就发散了。（降低non-object grid cells在loss function中所占weight）

3.sum squared error也equally weights large boxes和small boxes。但是实际上相同的deviation对于large box和small box的影响是不同的。（使用开根号来减轻这种差异）

由上，本文提出了一种新的loss function：

$L=\lambda\_{coord} \sum\_{i=0}^{S^2} \sum\_{j=0}^{B}1\_{ij}^{obj}[(x\_{ij}-\hat{x\_{ij}})^2+(y\_{ij}-\hat{y\_{ij}})^2] \\\ \quad +\lambda\_{coord} \sum\_{i=0}^{S^2} \sum\_{j=0}^{B} 1\_{ij}^{obj} [(\sqrt{w\_{ij}}-\sqrt{\hat{w\_{ij}}})^2+(\sqrt{h\_{ij}}-\sqrt{\hat{h\_{ij}}})^2] \\\ \quad +\sum\_{i=0}^{S^2} \sum\_{j=0}^{B} 1\_{ij}^{obj} (C\_{ij} - \hat{C\_{ij}})^2 \\\ \quad +\lambda\_{noobj}\sum\_{i=0}^{S^2} \sum\_{j=0}^{B}1\_{ij}^{noobj} (C\_{ij}-\hat{C\_{ij}})^2 \\\ \quad + \sum\_{i=0}^{S^2} 1\_{i}^{obj} \sum\_{c\in classes} (p\_i ( c )-\hat{p\_i ( c )})^2$

在本文中，$\lambda\_{coord}=5, \quad\lambda\_{noobj}=0.5$。

## Limitations

1.impose strong spatial constraints on bounding boxes. 因为每个grid cell只能预测2个boxes，和1个class。这样就限制了YOLO这个模型能够预测的相邻物体的数量。同时，也使得这个模型难以检测成群出现的小物体；

2.难以generalize to训练时没见过的aspect ratios或configurations；

3.该架构有许多downsampling layers，所以该模型使用了很多相对coarse的特征；

4.尽管使用了开根号减轻相同error对不同大小的object的影响差异，但是这种差异仍然存在，并且对结果会有较大影响。

## 参考

[YOLO v1深入理解](https://www.jianshu.com/p/cad68ca85e27)

[图解YOLO](https://zhuanlan.zhihu.com/p/24916786)

[从YOLOv1到YOLOv3，目标检测的进化之路](https://blog.csdn.net/guleileo/article/details/80581858)



